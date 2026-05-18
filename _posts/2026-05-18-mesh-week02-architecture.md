---
layout: post
title: "Mesh Week 2: Istio 아키텍처 — Control Plane, Data Plane, Envoy"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio envoy platform senior-series
---

## 학습 목표

- istiod 단일 control plane의 책임.
- Envoy의 5가지 listener/cluster/route/endpoint/secret (xDS).
- IPTables 리다이렉션.
- Sidecar injection 메커니즘.

## 1. 비유 — "오케스트라 지휘자 + 연주자"

istiod = 지휘자 (control plane). 모든 연주자(Envoy)에게 악보 배포 + 변경 알림.
Envoy = 연주자 (data plane). 실제 트래픽 처리.

## 2. istiod

단일 binary, 여러 역할:
- **Pilot**: Envoy 설정 (route, cluster, listener) 생성/배포.
- **Citadel**: CA, 인증서 발급.
- **Galley**: config 검증.

각 Envoy에 xDS gRPC stream으로 push.

## 3. Envoy

C++ 고성능 proxy. Lyft 원천. CNCF graduated.

기능:
- L4/L7 proxy.
- HTTP/1.1, HTTP/2, gRPC.
- TLS.
- Load balancing (round-robin, least-conn, ring hash).
- Retry, circuit breaker.
- Observability (access log, metric, trace).
- Filter chain (확장 가능).

자세한 Envoy 내부는 [보충 1: Envoy 깊게].

## 4. xDS Protocol

istiod ↔ Envoy 통신. 5가지 자원:

- **LDS** (Listener): 어떤 포트·프로토콜 listen.
- **RDS** (Route): HTTP 경로 라우팅 규칙.
- **CDS** (Cluster): upstream service 그룹.
- **EDS** (Endpoint): cluster의 실제 IP:port.
- **SDS** (Secret): TLS cert.

```
istiod → [LDS]  → "listen on 0.0.0.0:9080"
       → [RDS]  → "path /api/* → cluster productpage"
       → [CDS]  → "cluster productpage uses round-robin"
       → [EDS]  → "productpage endpoints: 10.0.0.5, 10.0.0.6"
       → [SDS]  → "TLS cert: <PEM>"
```

## 5. IPTables Redirection

Pod의 모든 outgoing/incoming TCP가 Envoy를 거치도록 init container가 iptables 규칙 설정:

```
[App] → outbound → iptables redirect → Envoy (15001) → external
external → iptables redirect → Envoy (15006) → App
```

App은 자기가 mesh 안에 있는지 모름. transparent.

## 6. Sidecar Injection

### Automatic
namespace에 라벨:
```bash
kubectl label namespace default istio-injection=enabled
```

이후 Pod 생성 시 mutating webhook이 istio-proxy 컨테이너 + init container 자동 주입.

### Manual
```bash
istioctl kube-inject -f deployment.yaml | kubectl apply -f -
```

CI 단계에서 inject.

## 7. Installation Profiles

```bash
istioctl install --set profile=demo
```

profile:
- **default**: production 표준.
- **demo**: 데모용 (모든 기능 on).
- **minimal**: control plane만.
- **ambient**: sidecar 없는 모드.

operator 형태로도 설치:
```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: default
```

## 8. 메트릭 & UI

- istiod metric → Prometheus.
- Kiali → mesh 시각화.
- Jaeger → trace.

6주차 깊게.

## 9. 운영 함정 5선

1. control plane HA 부재.
2. injection 누락 (라벨 미설정).
3. iptables가 다른 sidecar(다른 mesh) 와 충돌.
4. xDS 부하 (수천 Pod) → istiod scale.
5. 잘못된 profile 사용.

## 10. 자가평가 퀴즈

### Q1. istiod 책임?
1. **Pilot/Citadel/Galley 통합 - Envoy 설정 push, CA, 검증**
2. data plane
3. UI
4. 무관

**정답: 1.**

### Q2. Envoy의 5가지 xDS?
1. **LDS/RDS/CDS/EDS/SDS**
2. UI
3. 무관
4. 4가지

**정답: 1.**

### Q3. App이 mesh 안에 있는지 아는가?
1. 아니다 (iptables redirection으로 transparent)
2. 알고 있음
3. UI로 확인
4. 무관

**정답: 1.**

### Q4. Automatic injection 방법?
1. **namespace label `istio-injection=enabled`**
2. 모든 Pod 수동
3. UI
4. 무관

**정답: 1.**

### Q5. Profile demo는?
1. **데모용 (모든 기능 on, production 부적합)**
2. production 표준
3. UI 강함
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 3: Sidecar injection + VirtualService]에서 실제 트래픽 제어 시작.
