---
layout: post
title: "O11y 보충 2: OpenTelemetry Collector Pipeline 설계"
date: 2026-05-18 18:00:00 +0900
categories: observability opentelemetry platform senior-series supplement
---

8주차 OTel collector를 pipeline 설계 수준으로.

## 1. Pipeline 3 부

```
Receivers → Processors → Exporters
```

각 신호(metric/log/trace)마다 별도 pipeline.

## 2. Receivers

- `otlp` (gRPC/HTTP): SDK가 직접 push.
- `prometheus`: scrape (Prometheus 호환).
- `filelog`: 파일 tail (log).
- `jaeger`, `zipkin`: 옛 trace 호환.
- `fluentforward`: Fluent Bit 호환.

## 3. Processors

### batch
보내기 전 모음. 효율 + 비용.

### memory_limiter
collector 자체 OOM 방지.

### attributes / resource
attribute 추가/삭제/변경.

### filter
조건 안 맞는 데이터 drop.

### tail_sampling (traces)
tail-based sampling.

### transform
복잡 변환 (OTTL DSL).

### k8sattributes
K8s metadata 자동 추가 (pod.name, namespace 등).

### probabilistic_sampler
head sampling.

## 4. Exporters

- `prometheusremotewrite`: Prometheus 호환 remote write.
- `loki`: Loki push.
- `otlp` (tempo/jaeger): trace.
- `kafka`: 메시지 큐.
- `awsxray`, `googlecloud`, `azuremonitor`: cloud SaaS.
- `file`: 디버깅용.

## 5. 실전 Pipeline 예시

### Metrics
```yaml
receivers: [otlp, prometheus]
processors: [memory_limiter, batch, attributes, resource]
exporters: [prometheusremotewrite/thanos]
```

### Logs
```yaml
receivers: [otlp, filelog]
processors: [memory_limiter, batch, k8sattributes]
exporters: [loki]
```

### Traces
```yaml
receivers: [otlp]
processors: [memory_limiter, batch, tail_sampling, k8sattributes]
exporters: [otlp/tempo]
```

## 6. Agent vs Gateway

### Agent (per node DaemonSet)
- 노드 local Pod의 telemetry 수집.
- 가벼움 (CPU/메모리 적게).
- batch + 간단 enrich만.

### Gateway (cluster Deployment)
- agent들의 데이터 통합.
- 무거운 처리 (tail_sampling, complex transform).
- HA + autoscale.

빅테크 표준: agent + gateway 2층.

## 7. Tail Sampling

```yaml
tail_sampling:
  decision_wait: 30s
  num_traces: 100000
  policies:
    - { name: errors, type: status_code, status_code: { status_codes: [ERROR] } }
    - { name: slow, type: latency, latency: { threshold_ms: 5000 } }
    - { name: rare, type: rate_limiting, rate_limiting: { spans_per_second: 100 } }
```

error는 100%, slow는 100%, 나머지는 rate limited.

## 8. 운영 함정

1. batch interval 너무 길게 → 메모리 OOM.
2. memory_limiter 없음 → cluster 위험.
3. k8sattributes RBAC 부재 → metadata 없음.
4. tail_sampling memory 부족.
5. exporter queue 무제한 → backpressure 없음.

## 9. 결론

collector pipeline 설계가 모든 빅테크 o11y의 중심. 잘 설계하면 vendor·backend 자유, 비용 효율, 디버깅 풀.
