---
layout: post
title: "O11y Week 8: OpenTelemetry — 단일 Instrumentation으로 3종 Backend"
date: 2026-05-18 18:00:00 +0900
categories: observability opentelemetry platform senior-series
---

## 학습 목표

- OTel의 3 컴포넌트 (API, SDK, Collector).
- 단일 instrumentation으로 metric/log/trace 동시 송출.
- Collector pipeline 설계.
- Vendor-neutral의 의의.

## 1. 비유 — "다국어 어댑터"

비유: 한 가게가 영어/한국어/일본어 손님을 받습니다. 각 언어 통역사를 따로 두면 직원이 너무 많음. 한 명의 "다국어 어댑터"가 모든 통역. → OTel이 그 어댑터.

App은 OTel API만 호출. 출력은 Prometheus/Tempo/Loki/Datadog/NewRelic 등 무엇이든 가능.

## 2. 3 컴포넌트

### API
표준 interface. `Tracer`, `Meter`, `Logger`. 언어별 (Go, Python, Java, JS, .NET, ...).

### SDK
API 구현체. exporter 연결, sampling 등 설정.

### Collector
별도 process. 받은 데이터를 변환/필터링 후 backend로.

## 3. Instrument 예시 (Go)

### Trace
```go
tracer := otel.Tracer("myapp")
ctx, span := tracer.Start(ctx, "operation")
defer span.End()
span.SetAttributes(attribute.String("user.id", "alice"))
```

### Metric
```go
meter := otel.Meter("myapp")
counter, _ := meter.Int64Counter("requests_total")
counter.Add(ctx, 1, metric.WithAttributes(attribute.String("status", "200")))

histo, _ := meter.Float64Histogram("request_duration_seconds")
histo.Record(ctx, 0.123)
```

### Log (newer)
```go
logger := otelglobal.Logger("myapp")
logger.InfoContext(ctx, "request processed", "user_id", "alice")
```

## 4. Auto-instrumentation

언어별 agent로 코드 수정 없이 자동.

- Java: Java agent jar.
- Python: opentelemetry-instrument CLI.
- Node: --require @opentelemetry/auto-instrumentations-node.
- Go: 일부 라이브러리만 (수동 권장).

## 5. Collector

```yaml
receivers:
  otlp:
    protocols:
      grpc: {}
      http: {}
  prometheus:
    config: ...
processors:
  batch: {}
  memory_limiter:
    limit_mib: 1024
  tail_sampling:
    decision_wait: 30s
    policies:
      - { name: errors, type: status_code, status_code: { status_codes: [ERROR] } }
      - { name: slow, type: latency, latency: { threshold_ms: 5000 } }
  attributes:
    actions:
      - key: env
        value: prod
        action: insert
exporters:
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
  otlp/tempo:
    endpoint: tempo:4317
service:
  pipelines:
    metrics: { receivers: [otlp], processors: [batch], exporters: [prometheusremotewrite] }
    logs:    { receivers: [otlp], processors: [batch], exporters: [loki] }
    traces:  { receivers: [otlp], processors: [tail_sampling, batch], exporters: [otlp/tempo] }
```

자세한 collector pipeline은 [보충 2].

## 6. Deployment 패턴

- **Agent (DaemonSet)**: 각 노드에 collector. local Pod의 telemetry 수집.
- **Gateway (Deployment)**: 중앙 collector. 가공·집계 후 backend로.

빅테크 표준: Agent + Gateway 둘 다.

```
[Pod] → [Agent DaemonSet (per node)] → [Gateway (cluster)] → backends
```

## 7. Vendor-neutral의 의의

OTel 이전: vendor lock-in. Datadog 쓰면 Datadog SDK, NewRelic 쓰면 NewRelic SDK. 변경 비용 크.

OTel 이후: 한 번 instrument, vendor는 collector exporter만 변경.

빅테크는 SaaS vendor 비용 절감, multi-vendor 자유.

## 8. 표준 Semantic Convention

attribute 이름 표준화. 예:
- `http.method`, `http.status_code`
- `db.system`, `db.statement`
- `k8s.pod.name`, `k8s.namespace.name`
- `service.name`, `service.version`

이걸 따르면 모든 vendor가 같은 query/dashboard.

## 9. 운영 함정 5선

1. SDK 버전 분산 → 호환 문제.
2. Collector OOM (resource limit 부재).
3. attribute cardinality 폭발 → Prometheus 영향.
4. semantic convention 무시 → vendor 별 dashboard 깨짐.
5. agent vs gateway 안 나누고 단일 → scale 한계.

## 10. 빅테크 사례

### Adobe
OTel 표준화 후 vendor 자유. 비용 절감.

### Shopify
OTel + Honeycomb. tail-sampling으로 비용 효율.

### Microsoft
OTel 적극 contributor. Azure Monitor가 OTel 표준 지원.

## 11. 실습

```bash
# 1. OTel Operator 설치
# 2. Demo app (otel astronomy-shop)
# 3. Instrumentation auto 적용
# 4. Collector pipeline 3종 backend
# 5. tail-sampling으로 error만 보관
```

## 12. 자가평가 퀴즈

### Q1. OTel 3 컴포넌트?
1. **API, SDK, Collector**
2. UI/CLI/Library
3. 무관
4. trace/metric/log

**정답: 1.**

### Q2. Auto-instrumentation의 가치?
1. **코드 수정 없이 자동 trace/metric**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q3. Agent + Gateway 분리?
1. **각 노드 수집 + 중앙 가공/집계 — scale**
2. 같음
3. UI
4. 무관

**정답: 1.**

### Q4. Vendor-neutral 의의?
1. **vendor 변경 비용 0 — collector exporter만 교체**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. Semantic convention 무시?
1. **vendor별 dashboard 깨짐**
2. 안전
3. 빠름
4. 무관

**정답: 1.**

## 13. 다음 주차

[Week 9: Grafana]에서 dashboards 운영.
