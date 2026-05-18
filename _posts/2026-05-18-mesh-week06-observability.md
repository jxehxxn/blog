---
layout: post
title: "Mesh Week 6: 관측 — Kiali, Jaeger, Prometheus 통합"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh istio observability platform senior-series
---

## 학습 목표

- Istio의 자동 telemetry (RED metric 자동).
- Kiali로 service graph 시각화.
- Jaeger/Tempo trace 자동.
- access log 활용.

## 1. 자동 RED Metrics

Envoy가 모든 L7 호출의 다음 metric 자동 생성:
- `istio_requests_total` (counter, code/method/source/destination)
- `istio_request_duration_milliseconds` (histogram)
- `istio_request_bytes`, `istio_response_bytes`

앱 instrumentation 0으로 RED 완성.

```promql
# 5xx 비율
sum(rate(istio_requests_total{response_code=~"5.."}[5m]))
/
sum(rate(istio_requests_total[5m]))

# p99 latency
histogram_quantile(0.99,
  sum(rate(istio_request_duration_milliseconds_bucket[5m])) by (le, destination_service_name))
```

## 2. Kiali — Service Graph

설치:
```bash
kubectl apply -f samples/addons/kiali.yaml
```

UI 기능:
- Service topology (실시간).
- RED metric overlay.
- VirtualService/DestinationRule edit.
- 인증/인가 정책 시각화.

빅테크 디버깅: mesh의 첫 도구.

## 3. Jaeger / Tempo — Trace

Envoy가 trace를 자동 생성 (parent-child relationship 자동). collector로 push.

app은:
- trace context (b3 또는 traceparent header) propagate만 하면 됨.
- "내가 호출했다" 만 알리면 Envoy가 trace 생성.

```bash
kubectl apply -f samples/addons/jaeger.yaml
```

UI: trace search, span 시각화, 분산 latency 분석.

## 4. Access Log

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata: { name: access-log, namespace: istio-system }
spec:
  accessLogging:
    - providers: [{ name: envoy }]
```

모든 요청 log:
```
[2026-05-18T14:00:00Z] "GET /api/v1/charges HTTP/1.1" 200 - via_upstream - "-" 0 256 12 12 "-"
  "Mozilla/5.0..." "abc-123" "payments-api.payments.svc.cluster.local" "10.0.0.5:8080"
```

Fluent Bit으로 Loki 적재.

## 5. Telemetry API

v1.15+ Telemetry CR로 세밀 제어:

```yaml
spec:
  metrics:
    - providers: [{ name: prometheus }]
      overrides:
        - match: { metric: REQUEST_COUNT }
          tagOverrides:
            user_id: { operation: REMOVE }   # cardinality 줄임
  tracing:
    - providers: [{ name: jaeger }]
      randomSamplingPercentage: 1.0   # 1%
  accessLogging:
    - providers: [{ name: envoy }]
      filter: { expression: "response.code >= 400" }   # 4xx/5xx만
```

## 6. Sampling 정책

trace 100%면 비용 폭증. 보통 1~5%. 7주차 (o11y 강의 보충 4) 와 동일.

## 7. Kiali와 Grafana 결합

Grafana datasource로 Prometheus (Istio metric). 표준 dashboard import:
```bash
istioctl dashboard grafana
```

## 8. 운영 함정 5선

1. Telemetry CR 미설정 → 기본 100% trace (비용).
2. user_id 등 label 자동 추가 → cardinality 폭발.
3. access log 100% → 디스크.
4. Kiali에 prod 권한 부여 → 변경 위험.
5. Telemetry 적용 cluster-wide → 의도와 다른 effect.

## 9. 실습

```bash
# 1. Istio + Kiali + Jaeger + Prometheus 설치
# 2. Bookinfo traffic으로 service graph 확인
# 3. Telemetry CR로 trace 1% sampling
# 4. PromQL로 RED 메트릭
# 5. trace 클릭 → log correlation
```

## 10. 자가평가 퀴즈

### Q1. Istio 자동 metric?
1. **RED 자동 (requests_total, duration, bytes)**
2. app instrumentation 필요
3. UI만
4. 무관

**정답: 1.**

### Q2. Kiali 가치?
1. **service graph 실시간 시각화**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q3. Trace context propagation?
1. **app이 header만 propagate, Envoy가 span 생성**
2. app이 모두
3. UI
4. 무관

**정답: 1.**

### Q4. Telemetry CR 가치?
1. **세밀 제어 (metric 변환, sampling 비율, log filter)**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. Trace 100% 의 문제?
1. **비용 폭증, collector 부담**
2. 안전
3. 빠름
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 7: Gateway + Ingress + Egress]에서 cluster 경계 트래픽.
