---
layout: post
title: "Mesh Week 3: VirtualService, DestinationRule, Sidecar Injection 실전"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio platform senior-series
---

## 학습 목표

- VirtualService로 트래픽 라우팅.
- DestinationRule로 subset + LB 정책.
- Sidecar 리소스 사용 최적화.
- Bookinfo 데모로 한 번 흐름 보기.

## 1. 4종 핵심 CRD

- `VirtualService`: 어떤 트래픽이 어디로 갈지 (route).
- `DestinationRule`: 도착 후 어떻게 다룰지 (subset, LB, TLS).
- `Gateway`: cluster 외부 진입점 (다음 주).
- `ServiceEntry`: 외부 서비스를 mesh에 등록.

## 2. VirtualService 예시

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata: { name: productpage }
spec:
  hosts: [productpage]
  http:
    - match:
        - headers:
            user-agent:
              regex: ".*Mobile.*"
      route:
        - { destination: { host: productpage, subset: mobile } }
    - route:
        - { destination: { host: productpage, subset: default }, weight: 90 }
        - { destination: { host: productpage, subset: v2 }, weight: 10 }
```

- mobile user → mobile subset.
- 그 외 90% default, 10% v2 (canary).

## 3. DestinationRule 예시

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata: { name: productpage }
spec:
  host: productpage
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      tcp: { maxConnections: 100 }
      http: { http2MaxRequests: 1000, maxRequestsPerConnection: 10 }
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
    - name: default
      labels: { version: v1 }
    - name: v2
      labels: { version: v2 }
    - name: mobile
      labels: { version: mobile }
```

- LB: least connections.
- Connection pool 제한.
- Outlier detection: 5번 연속 5xx면 30초 ejection.

## 4. Bookinfo 실습

Istio 표준 데모.

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

UI에 productpage → details + reviews → ratings 호출. 4 서비스.

```bash
# 100% reviews v1
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml

# 20% v2, 80% v1 canary
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

trafficshift를 yaml 한 줄로.

## 5. Sidecar Resource

namespace 단위로 Envoy 메모리·CPU 최적화:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata: { name: default, namespace: payments }
spec:
  egress:
    - hosts:
        - "./*"             # 같은 namespace
        - "kube-system/*"    # kube-system
```

Envoy가 전 cluster service 정보를 가지면 메모리 폭증. Sidecar로 필요 부분만.

빅테크 표준: 모든 namespace에 Sidecar resource 두어 Envoy 메모리 200MB → 30MB.

## 6. Service Entry

외부 서비스를 mesh에 등록:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata: { name: github-api }
spec:
  hosts: [api.github.com]
  ports:
    - { number: 443, name: https, protocol: HTTPS }
  resolution: DNS
  location: MESH_EXTERNAL
```

이후 VirtualService로 timeout/retry 가능.

## 7. PeerAuthentication (5주차 미리)

mTLS 강제:
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata: { name: default, namespace: istio-system }
spec:
  mtls: { mode: STRICT }
```

## 8. 실습

```bash
# 1. Bookinfo 설치
# 2. VirtualService로 90/10 canary
# 3. DestinationRule subset 정의
# 4. outlierDetection 동작 확인 (의도적 5xx)
# 5. Sidecar resource로 메모리 비교
```

## 9. 자가평가 퀴즈

### Q1. VirtualService 책임?
1. **트래픽 라우팅 (어디로)**
2. LB
3. 인증
4. 무관

**정답: 1.**

### Q2. DestinationRule 책임?
1. **도착 후 처리 (subset, LB, connection pool, outlier)**
2. 라우팅
3. UI
4. 무관

**정답: 1.**

### Q3. Outlier detection 효과?
1. **연속 실패 endpoint 일시 ejection — circuit breaker**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q4. Sidecar resource의 가치?
1. **Envoy 메모리 최적 — 필요한 service만 인지**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. ServiceEntry 용도?
1. **외부 service를 mesh에 등록 — timeout/retry 적용**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 10. 다음 주차

[Week 4: Traffic Management 깊게]에서 mirror, fault injection, retry를.
