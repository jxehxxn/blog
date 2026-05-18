---
layout: post
title: "O11y Week 3: PromQL 깊게 — Aggregation, Histogram, Window"
date: 2026-05-18 18:00:00 +0900
categories: observability prometheus platform senior-series promql
---

## 학습 목표

- rate/increase/irate의 정확한 차이.
- histogram_quantile로 p99 계산.
- 다양한 aggregator (sum/avg/max/topk/bottomk).
- offset, subquery, binary operator.

## 1. rate vs increase vs irate

- `rate(metric[5m])`: 5분 윈도우의 **초당 평균** 증가율.
- `increase(metric[5m])`: 5분 동안 **총** 증가량 (= rate × 300).
- `irate(metric[5m])`: 마지막 2개 샘플 기준 **순간** rate.

빅테크 표준: 일반 dashboard에는 rate, spike 분석엔 irate.

## 2. Histogram 활용

```promql
# p95 latency
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

핵심:
- `_bucket` 시리즈를 rate로 → 분포 추세.
- `sum by (le)` 로 모든 instance 합산.
- `histogram_quantile(0.95, ...)` 로 quantile 추론.

`by (le)`를 빠뜨리면 결과가 비어 나옴 (학부생 흔한 함정).

## 3. Aggregation Operators

```promql
sum(rate(metric[5m]))           # 전체 합
sum by (status) (rate(metric))  # status별
avg by (instance) (cpu_usage)
max by (job) (memory)
min(...)
count(...)
stddev(...)
stdvar(...)
topk(3, rate(metric[5m]))       # 상위 3개
bottomk(3, ...)
quantile(0.99, ...)
```

`by` 와 반대로 `without` 도 있음: `sum without (instance) (...)` = instance 제외하고 합.

## 4. Binary Operators

```promql
# 비율
sum(rate(http_requests_total{code=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# 임계 비교
node_memory_MemAvailable_bytes < 100*1024*1024
```

label matching:
- `on(...)`: 매칭에 사용할 label.
- `ignoring(...)`: 매칭에서 무시할 label.
- `group_left()`, `group_right()`: 1:N matching.

## 5. Window / Subquery

5분 평균을 1시간 동안 매분 평가:
```promql
avg_over_time(rate(metric[5m])[1h:1m])
```

`[1h:1m]` 형식의 subquery.

다른 over_time 함수: `min_over_time`, `max_over_time`, `sum_over_time`, `count_over_time`, `quantile_over_time`.

## 6. Offset — 과거 시점

```promql
http_requests_total offset 1d
```

24시간 전 값. 비교에 유용:
```promql
rate(metric[5m]) / rate(metric[5m] offset 1d)
```

## 7. Time-shift Alerting

```yaml
- alert: WeekOverWeekDrop
  expr: |
    rate(orders_total[1h])
    /
    rate(orders_total[1h] offset 7d) < 0.7
  for: 10m
```

지난주 같은 시간 대비 30% 이상 감소.

## 8. 실전 PromQL 모음

```promql
# RED method (Rate/Errors/Duration)
# Rate
sum(rate(http_requests_total{job="app"}[5m]))
# Errors
sum(rate(http_requests_total{job="app",code=~"5.."}[5m]))
# Duration p99
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# USE method (Utilization/Saturation/Errors)
# CPU util
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
# memory saturation
container_memory_working_set_bytes / container_spec_memory_limit_bytes

# K8s pod restart
increase(kube_pod_container_status_restarts_total[1h]) > 0
```

## 9. PromQL 함정 5선

1. **rate에 gauge**: rate는 counter용. gauge엔 `delta()`.
2. **histogram_quantile without `by (le)`**: 결과 빈 값.
3. **`sum(rate(...))` 결과를 `instant_vector`로 가정**: 시점 vs 윈도우 헷갈림.
4. **`/` 의 zero division**: `or vector(0)` 같은 fallback.
5. **단일 quantile 의존**: p99만 보지 말고 p50/p95도.

## 10. 실습

다음 PromQL 직접 작성:
1. 지난 5분 99 percentile latency (job별).
2. 5xx 비율 1% 넘으면 alert.
3. 메모리 사용량이 1주 전 같은 시각 대비 50% 증가 alert.
4. top 5 pod by CPU.
5. pod restart 1시간 내 1회 이상.

## 11. 자가평가 퀴즈

### Q1. rate vs increase?
1. **rate 초당, increase 총량**
2. 같음
3. 반대
4. 무관

**정답: 1.**

### Q2. histogram_quantile에 빠뜨리면 안 되는 것?
1. **by (le)**
2. by (instance)
3. UI
4. 무관

**정답: 1.**

### Q3. topk(3, ...) 효과?
1. **상위 3개 시리즈**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q4. offset 1d 가치?
1. **24시간 전 값과 비교**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. RED method 의 R/E/D?
1. **Rate / Errors / Duration**
2. Red/Errors/Duration
3. 무관
4. UI

**정답: 1.**

## 12. 다음 주차

[Week 4: Alertmanager]에서 alert routing, grouping, silencing을 다룹니다.
