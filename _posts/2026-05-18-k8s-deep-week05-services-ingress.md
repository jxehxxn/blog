---
layout: post
title: "K8s Deep Week 5: Service, Ingress, Gateway API — 트래픽이 외부에서 Pod까지"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series networking
---

## 학습 목표

- Service 4종류(ClusterIP, NodePort, LoadBalancer, ExternalName)를 비유로 구분한다.
- Ingress controller가 L7 라우팅을 어떻게 처리하는지 안다.
- 차세대 Gateway API가 Ingress와 무엇이 다른지 안다.
- cert-manager로 TLS 자동 발급을 한다.

## 1. Service — "변하지 않는 우편함"

비유: 강의실(Pod)은 학기마다 바뀝니다(Pod 재시작 시 IP 바뀜). 그런데 학생은 "교수 A님 강의" 라는 고정된 주소로 편지를 보내고 싶습니다. Service는 그 "고정 우편함" 입니다.

### 4종류

#### (1) ClusterIP (기본)
클러스터 안에서만 접근 가능한 가상 IP.

```yaml
apiVersion: v1
kind: Service
metadata: { name: webapp }
spec:
  selector: { app: webapp }
  ports:
    - port: 80
      targetPort: 8080
```

비유: 캠퍼스 안의 학생들만 쓰는 내선 번호.

#### (2) NodePort
모든 노드의 같은 포트에서 접근 가능 (예: 노드 IP:30080).

비유: 캠퍼스 정문마다 같은 번호의 외부 우편함 설치.

#### (3) LoadBalancer
클라우드의 외부 로드밸런서를 자동 생성.

비유: 캠퍼스 외부에 큰 콜센터를 두고, 모든 외부 전화가 콜센터를 거쳐 캠퍼스 내선으로 연결.

#### (4) ExternalName
DNS CNAME 만들어 외부 호스트를 가리킴.

비유: 캠퍼스 전화번호부에 "교수 X님 = 010-...-..." 처럼 외부 번호를 등록.

## 2. Service가 어떻게 Pod을 찾나 — Endpoint

비유: 우편함과 강의실을 매핑하는 표가 따로 있습니다. 강의실이 바뀌면 그 표만 갱신.

```bash
kubectl get svc webapp
# CLUSTER-IP   10.96.0.10

kubectl get endpoints webapp
# ENDPOINTS    10.244.0.5:8080,10.244.0.6:8080,10.244.0.7:8080
```

**Endpoint controller**가 selector 매칭되는 Pod IP를 Endpoint 객체로 동기화. kube-proxy가 이 Endpoint를 iptables 규칙으로 변환.

K8s v1.21+ 에서는 **EndpointSlice** (Endpoint의 샤딩 버전)가 표준. 큰 Service(1000+ Pod)도 성능 ok.

## 3. Headless Service — 학부생 함정

`clusterIP: None` 으로 만들면 가상 IP 없이 **DNS로 모든 Pod IP를 직접 반환**.

```bash
nslookup webapp-headless
# 10.244.0.5
# 10.244.0.6
# 10.244.0.7
```

용도: StatefulSet(예: Cassandra, Kafka) 처럼 클라이언트가 직접 Pod IP를 알아야 할 때.

## 4. Ingress — "L7 라우터"

비유: 콜센터(LoadBalancer)는 전화를 받지만 어디로 연결할지 모릅니다. **Ingress**는 "안녕하세요로 시작하면 한국어팀, hello로 시작하면 영어팀" 같은 L7 규칙(HTTP path/host)으로 트래픽을 분배.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [app.mycorp.com]
      secretName: app-tls
  rules:
    - host: app.mycorp.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: { name: api, port: { number: 80 } }
          - path: /
            pathType: Prefix
            backend:
              service: { name: webapp, port: { number: 80 } }
```

### Ingress Controller — 핵심 분리

학부생이 자주 헷갈리는 것: **Ingress 객체는 명세이고, 실제로 일하는 건 Ingress Controller(NGINX/Traefik/HAProxy 등)**.

```
[외부 트래픽]
  ↓
[LoadBalancer (클라우드 LB)]
  ↓
[Ingress Controller Pod (NGINX 등)]
  ↓ Ingress 규칙대로 분배
[Service (ClusterIP)]
  ↓
[Pod]
```

Ingress 객체만 만들고 Controller를 설치 안 하면 아무 일도 안 일어남.

## 5. Gateway API — "차세대 Ingress"

Ingress의 한계:
- 한 객체에 모든 규칙 몰림 → 멀티팀 협업 어려움.
- 표현력 부족 (path/host만, TCP/UDP/gRPC 약함).
- vendor 별 annotation 난립.

**Gateway API**는 이를 해결:
- `GatewayClass`: 인프라팀이 정의 (NGINX, Istio, Envoy 등).
- `Gateway`: 플랫폼팀이 진입점 정의 (포트, hostname, TLS).
- `HTTPRoute`/`TCPRoute`/`GRPCRoute`: 앱팀이 라우팅 규칙 정의.

비유: 빌딩 관리(GatewayClass) ↔ 층 운영(Gateway) ↔ 사무실 안 작업(Route)으로 책임이 분리.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: { name: prod-gateway }
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls: { certificateRefs: [{name: prod-tls}] }
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: api-route, namespace: payments }
spec:
  parentRefs: [{name: prod-gateway, namespace: gateway-system}]
  hostnames: [api.mycorp.com]
  rules:
    - matches: [{ path: { type: PathPrefix, value: /payments } }]
      backendRefs: [{ name: payments-svc, port: 80 }]
```

빅테크 추세: Ingress → Gateway API 전환 중.

## 6. cert-manager — TLS 자동화

비유: 매 학기 인증서를 우체국에서 받아오는 일을 자동화. **Let's Encrypt** 같은 무료 CA에서 자동 발급/갱신.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: { name: letsencrypt-prod }
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: secops@mycorp.com
    privateKeySecretRef: { name: letsencrypt-account-key }
    solvers:
      - http01:
          ingress: { class: nginx }
```

Ingress에 annotation 한 줄(`cert-manager.io/cluster-issuer: letsencrypt-prod`)로 자동.

빅테크 표준: 내부 CA + Vault PKI + cert-manager Vault issuer. 외부 도메인은 Let's Encrypt.

## 7. external-dns

비유: 강의실이 바뀔 때마다 캠퍼스 전화번호부를 자동으로 갱신.

`external-dns`가 K8s Service/Ingress의 hostname을 보고 Route53/Cloud DNS/Cloudflare에 자동으로 A record 등록.

## 8. 실습

```bash
# 1. NGINX Ingress Controller 설치
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace

# 2. 두 앱 (api, web) 배포 + 각각 Service
# 3. Ingress 만들어서 /api → api, / → web 라우팅
# 4. cert-manager 설치 후 Let's Encrypt로 TLS 발급
# 5. Gateway API 변환해 같은 라우팅 재구성
```

## 9. 자가평가 퀴즈

### Q1. ClusterIP 와 NodePort의 차이?
1. ClusterIP 클러스터 내부, **NodePort는 노드 IP:30000~32767로 외부 접근 가능**
2. 같은 것
3. NodePort가 더 안전
4. 무관

**정답: 1.**

### Q2. EndpointSlice 가 Endpoint를 대체한 이유?
1. **큰 Service(수천 Pod)에서 성능·확장성**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q3. Ingress 객체만 만들고 Controller가 없으면?
1. **아무 일도 안 일어남**
2. 자동 설치
3. 오류
4. 무관

**정답: 1.**

### Q4. Gateway API가 Ingress보다 좋은 점?
1. **인프라/플랫폼/앱팀의 책임 분리, L7+L4 표현력**
2. UI
3. 무료
4. 빠름

**정답: 1.**

### Q5. cert-manager의 핵심 가치?
1. **TLS 인증서 자동 발급/갱신**
2. Pod 띄움
3. UI
4. 무관

**정답: 1.**

## 10. 다음 주차

[Week 6: Storage — PV/PVC/CSI]에서 영구 스토리지의 모든 것을 다룹니다.
