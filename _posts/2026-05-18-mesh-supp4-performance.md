---
layout: post
title: "Mesh 보충 4: Mesh Performance Tuning"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio performance platform senior-series supplement
---

10주차 운영을 성능 영역으로.

## 1. 병목 위치

1. istiod xDS push (수천 Pod).
2. Envoy 메모리 (catalog 크기).
3. mTLS handshake CPU.
4. Telemetry overhead (1~5% latency).

## 2. istiod 튜닝

```yaml
spec:
  pilot:
    autoscaleEnabled: true
    autoscaleMin: 2
    autoscaleMax: 10
    resources:
      requests: { cpu: 500m, memory: 2Gi }
      limits: { cpu: 4, memory: 16Gi }
    env:
      PILOT_PUSH_THROTTLE: 100
      PILOT_DEBOUNCE_AFTER: 100ms
      PILOT_DEBOUNCE_MAX: 10s
```

- Debounce: 짧은 시간 변경 합치기.
- Push throttle: 동시 push 한도.

## 3. Envoy 메모리

- Sidecar resource로 catalog 제한 (3주차).
- `concurrency=1` (보통). 단일 worker thread.
- ConfigDump 정기 모니터링 (`/config_dump`).

## 4. mTLS CPU

Handshake가 비싼 연산. 대책:
- TLS session resumption.
- Connection pool (DestinationRule).
- keepalive 활용.

큰 차이 없으면 그냥 충분.

## 5. Telemetry overhead

- trace sampling 1%.
- access log 4xx/5xx만.
- metric label cardinality 관리 (user_id 제거).

## 6. Latency 영향

평균 latency overhead: ~5ms (sidecar L7).
- HBONE/ambient: ~2~3ms.
- mTLS handshake (cold): ~50ms 1회 후 amortize.

서비스 latency p99 100ms 대비 5% 정도. 보통 수용 가능.

## 7. Throughput

Envoy는 매우 빠름. CPU bound가 아니면 throughput 영향 적음.

벤치마크: 단일 Pod, 100k RPS 까지 OK (수많은 빅테크 검증).

## 8. 큰 cluster 운영

10,000 Pod cluster:
- istiod replica 5+ + HPA.
- Sidecar resource 적극.
- Telemetry sampling.
- xDS 부하 모니터링.

## 9. 빅테크 사례

### Salesforce
30k Pod cluster의 Istio. 자체 contribute로 scale 개선.

### LinkedIn
Envoy 직접 사용 + 일부 mesh feature 사용.

## 10. 결론

기본 설정으로 1000 Pod까지 OK. 그 이상은 위 튜닝 필수. 시니어의 차이는 운영 메트릭 + 튜닝.
