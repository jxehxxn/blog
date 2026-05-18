---
layout: post
title: "O11y 보충 4: Distributed Tracing Sampling + Correlation"
date: 2026-05-18 18:00:00 +0900
categories: observability tempo opentelemetry platform senior-series supplement
---

7주차 tracing sampling을 결정 모델 수준으로.

## 1. Sampling 종류

### Head sampling
요청 시작 시점 결정. SDK level.

```python
# Probabilistic 1%
tracer = TracerProvider(sampler=ParentBased(TraceIdRatioBased(0.01)))
```

장점: cheap. 단점: error/slow trace 놓침.

### Tail sampling
trace 완료 후 결정. Collector level.

```yaml
tail_sampling:
  policies:
    - { name: errors, type: status_code, ... }
    - { name: slow, type: latency, ... }
```

장점: 흥미로운 것 다 보존. 단점: collector memory + decision wait.

### Adaptive sampling
부하에 따라 자동 조절. Datadog 등 SaaS.

## 2. 표준 패턴 (빅테크)

```
[SDK head sampling 100%]
       ↓
[Collector tail sampling]
   - errors 100%
   - slow (>5s) 100%
   - rare (low traffic endpoint) 100%
   - common: rate 1~5%
```

## 3. Decision Wait

tail sampling은 trace의 모든 span을 본 후 결정. trace 전체가 collector에 도착할 때까지 wait.

```yaml
tail_sampling:
  decision_wait: 30s
```

30초 이내 도착 안 한 span은 lost.

## 4. Collector Memory

100,000 trace × 평균 50 span × 1KB = 5GB. tail sampling용.

memory_limiter 필수.

## 5. Correlation

### Logs ↔ Traces
log에 trace_id 추가:
```python
log.info("processing", trace_id=trace.get_current_span().context.trace_id)
```

Loki/Grafana에서 클릭으로 trace 이동.

### Metrics ↔ Traces (Exemplars)
Prometheus histogram bucket에 sample trace_id:
```promql
http_request_duration_seconds_bucket{le="0.5"}  # bucket
# exemplar: trace_id="abc123"
```

Grafana panel에서 spike 클릭 → exemplar의 trace로.

### Traces ↔ Logs
trace span에서 관련 log 보기.

3 신호 통합 디버깅 → 빅테크 표준.

## 6. Trace Context Propagation

### W3C Trace Context
HTTP header:
```
traceparent: 00-<trace-id>-<span-id>-<flags>
tracestate: <vendor-specific>
```

OTel SDK 기본. Kubernetes Service Mesh가 자동 propagate.

### Baggage
임의 키-값을 trace 전반에 전달:
```
baggage: user.id=alice, request.priority=high
```

downstream service에서 사용.

## 7. Sampling 의사결정 가이드

| 상황 | 권장 |
|------|------|
| 저예산 | head 1% |
| Error 중요 | tail (error 100%) |
| 분산 시스템 디버깅 | tail |
| 100% 보관 | 위험 (비용/메모리) |

## 8. Sampling 함정

1. head 1% + tail policy 없음 → 99% 정보 손실.
2. parent-based sampling 안 함 → trace 일부만.
3. decision_wait 너무 짧음 → late span lost.
4. exemplar 미설정 → metric → trace 연결 안 됨.
5. trace_id를 log에 안 넣음 → log 검색 어려움.

## 9. 결론

sampling은 성능·비용·완성도의 균형. head + tail 결합 + correlation으로 빅테크 표준 운영.
