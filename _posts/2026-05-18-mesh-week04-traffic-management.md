---
layout: post
title: "Mesh Week 4: Traffic Management — Mirror, Fault Injection, Retry, Timeout"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio platform senior-series
---

## 학습 목표

- Mirror로 production 트래픽 shadow copy.
- Fault injection으로 chaos test.
- Retry/timeout 정책.
- Connection pool + circuit breaker.

## 1. Traffic Mirror — "녹화 카메라"

```yaml
spec:
  http:
    - route:
        - { destination: { host: payments, subset: v1 }, weight: 100 }
      mirror: { host: payments, subset: v2 }
      mirrorPercentage: { value: 100 }
```

- 100% 트래픽이 v1으로 (실제 응답).
- 동시에 v2에도 같은 요청 (응답은 버림).

용도: 새 버전을 실제 트래픽으로 검증하되 사용자 영향 0.

## 2. Fault Injection — "지진 훈련"

```yaml
spec:
  http:
    - fault:
        delay:
          percentage: { value: 10.0 }
          fixedDelay: 5s
        abort:
          percentage: { value: 5.0 }
          httpStatus: 500
      route:
        - destination: { host: ratings }
```

10% 요청에 5초 지연, 5% 요청에 500 응답. Chaos engineering.

용도: 다운스트림 장애에 시스템이 어떻게 반응하는지 확인.

## 3. Retry

```yaml
spec:
  http:
    - route:
        - { destination: { host: ratings } }
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,connect-failure,reset
```

5xx/connect-failure/reset 시 3회 재시도. 각 시도 timeout 2초.

함정: idempotent endpoint에만. POST 같은 mutating은 신중.

## 4. Timeout

```yaml
spec:
  http:
    - route:
        - { destination: { host: ratings } }
      timeout: 10s
```

전체 요청 (모든 retry 포함) 10초 초과 시 504.

## 5. Connection Pool & Circuit Breaker

```yaml
# DestinationRule
spec:
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
        maxRetries: 3
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

- TCP 최대 100 connection.
- 보류 100건 초과 시 즉시 503 (overload protection).
- 연속 5xx 5회면 endpoint ejection 30초.

이게 진짜 circuit breaker. app code 변경 0.

## 6. Header Routing

A/B 테스트:
```yaml
spec:
  http:
    - match:
        - headers:
            cookie:
              regex: "^(.*?;)?(user=beta)(;.*)?$"
      route:
        - { destination: { host: payments, subset: v2 } }
    - route:
        - { destination: { host: payments, subset: v1 } }
```

beta cookie 가진 user만 v2.

## 7. Locality Load Balancing

같은 region/zone 우선:
```yaml
spec:
  trafficPolicy:
    loadBalancer:
      localityLbSetting:
        enabled: true
        distribute:
          - from: us-east-1/*
            to:
              "us-east-1/*": 80
              "us-west-1/*": 20
```

us-east-1 client는 us-east-1 endpoint 80%, fallback us-west-1 20%.

빅테크 multi-region cost/latency 최적.

## 8. Sticky Session

```yaml
spec:
  trafficPolicy:
    loadBalancer:
      consistentHash:
        httpCookie:
          name: user
          ttl: 3600s
```

같은 user는 같은 endpoint. session affinity 필요한 legacy.

## 9. 운영 함정 5선

1. **retry on POST**: 중복 처리.
2. **mirror 100% + 외부 비용**: mirror도 호출되어 외부 의존성 2배.
3. **outlier ejection 너무 강함**: production 트래픽 fail.
4. **timeout > parent timeout**: 의미 없음.
5. **connection pool 미설정**: overload 시 cascading failure.

## 10. 실습

```bash
# 1. Bookinfo 환경
# 2. 90/10 canary
# 3. mirror로 shadow
# 4. fault injection으로 5% 500 → 시스템 반응
# 5. retry + timeout + circuit breaker 전체 적용
```

## 11. 자가평가 퀴즈

### Q1. Traffic Mirror 가치?
1. **production 트래픽으로 새 버전 검증, 사용자 영향 0**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q2. Fault Injection 용도?
1. **chaos engineering — 다운스트림 장애 시뮬레이션**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q3. Retry의 함정?
1. **idempotent 만 가능, POST 신중**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q4. Outlier detection 가치?
1. **연속 실패 endpoint 자동 ejection — circuit breaker**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q5. Locality LB 가치?
1. **같은 region/zone 우선 → latency/cost**
2. UI
3. 무관
4. 빠른 빌드

**정답: 1.**

## 12. 다음 주차

[Week 5: 보안]에서 mTLS, AuthorizationPolicy, JWT.
