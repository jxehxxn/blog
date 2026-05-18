---
layout: post
title: "O11y Week 2: Prometheus 기초 — Pull Model, Exporters, PromQL 첫걸음"
date: 2026-05-18 18:00:00 +0900
categories: observability prometheus platform senior-series
---

## 학습 목표

- Prometheus 설치 + 첫 scrape.
- 4가지 metric 타입(counter/gauge/histogram/summary).
- Service Discovery (K8s, file_sd, ec2_sd).
- 기본 PromQL.

## 1. 설치 (K8s)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kps prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

이걸로 Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics 한 번에 설치 (kube-prometheus-stack은 빅테크 표준 시작점).

UI:
```bash
kubectl -n monitoring port-forward svc/kps-prometheus 9090:9090
# http://localhost:9090
```

## 2. 4가지 Metric 타입

### Counter
단조 증가. 누적 카운트.
```
http_requests_total
```
사용: rate, increase로 처리.

### Gauge
오르락내리락. 현재 값.
```
memory_usage_bytes
```
사용: 직접 값 확인.

### Histogram
값의 분포 (bucket 단위).
```
http_request_duration_seconds_bucket{le="0.1"}
http_request_duration_seconds_bucket{le="0.5"}
http_request_duration_seconds_count
http_request_duration_seconds_sum
```
사용: histogram_quantile로 p50/p95/p99.

### Summary
client에서 직접 quantile 계산. Histogram보다 정확하나 aggregation 어려움.

## 3. Exporters

서비스가 직접 메트릭을 노출 못 할 때 옆에 두는 변환기.

- **node-exporter**: Linux 시스템 메트릭.
- **kube-state-metrics**: K8s 객체 상태.
- **blackbox-exporter**: 외부 endpoint probe.
- **mysql-exporter, redis-exporter** 등.

각 exporter가 `/metrics` 엔드포인트 제공.

## 4. Service Discovery

### K8s SD
```yaml
scrape_configs:
  - job_name: pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
```

Pod annotation `prometheus.io/scrape: "true"` 붙은 것만 자동 수집.

### kube-prometheus-stack 표준
ServiceMonitor / PodMonitor CRD로 declarative.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  labels: { release: kps }
spec:
  selector:
    matchLabels: { app: myapp }
  endpoints:
    - port: metrics
      interval: 30s
```

## 5. 기본 PromQL

### Instant query
현재 시점:
```promql
up
http_requests_total{job="myapp"}
node_memory_MemAvailable_bytes
```

### Range query
시간 범위:
```promql
rate(http_requests_total[5m])
```

`rate()`: counter의 초당 증가율.

### Aggregation
```promql
sum(rate(http_requests_total[5m])) by (status)
avg(node_memory_MemAvailable_bytes) by (instance)
```

3주차에서 깊게.

## 6. Recording Rules

자주 쓰는 query를 미리 계산해 저장:
```yaml
groups:
  - name: myapp
    rules:
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
```

Grafana에서 매번 계산 안 하고 미리 저장된 시계열 조회 → 빠름.

## 7. 빅테크 패턴 — Operator 기반

kube-prometheus-stack의 Prometheus Operator가 다음 CRD 제공:
- `Prometheus`: Prometheus 인스턴스.
- `ServiceMonitor`, `PodMonitor`: scrape target.
- `PrometheusRule`: recording + alerting rules.
- `Alertmanager`: AM 인스턴스.
- `AlertmanagerConfig`: AM 설정.

declarative → GitOps 자연스러움.

## 8. 운영 함정 5선

1. **Cardinality 폭발**: label 잘못 → OOM.
2. **Retention 너무 길게**: 디스크 폭증. 보통 15일, 장기 보관은 Thanos.
3. **Scrape interval 너무 짧음**: 30s 권장, 5s는 부담.
4. **PrometheusRule 안 쓰고 매번 PromQL**: 느림.
5. **HA 없이 단일 Prometheus**: 장애 시 메트릭 손실.

## 9. 실습

```bash
# 1. kube-prometheus-stack 설치
# 2. 임의 nginx 배포 후 nginx-prometheus-exporter 사이드카
# 3. ServiceMonitor 만들어 자동 수집
# 4. http_requests_total 등 4가지 metric 타입 확인
# 5. recording rule로 사전 계산
```

## 10. 자가평가 퀴즈

### Q1. 4가지 metric 타입?
1. **Counter, Gauge, Histogram, Summary**
2. CPU, Mem, Disk, Net
3. UI
4. 무관

**정답: 1.**

### Q2. Counter 처리?
1. **rate/increase로 변환**
2. 직접 값
3. UI
4. 무관

**정답: 1.**

### Q3. ServiceMonitor 가치?
1. **declarative scrape target — CRD로 GitOps 친화**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q4. Cardinality 폭발 흔한 원인?
1. **label에 고유값 (user_id, request_id)**
2. metric 수
3. UI
4. 무관

**정답: 1.**

### Q5. Recording rule 가치?
1. **자주 쓰는 query 사전 계산 → Grafana 빠름**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 3: PromQL 깊게]에서 aggregation/histogram/window function을 깊게.
