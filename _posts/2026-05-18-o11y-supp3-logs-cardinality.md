---
layout: post
title: "O11y 보충 3: Logs Cardinality + Cost 깊게"
date: 2026-05-18 18:00:00 +0900
categories: observability loki platform senior-series supplement
---

6주차 Loki cardinality를 비용 영역까지.

## 1. Cardinality 정의

Loki는 (label set) 마다 별도 stream. label cardinality = 가능한 label 조합 수.

10 namespace × 5 app × 2 env = 100 stream. OK.
10 namespace × 5 app × 1M users = 50M stream. 죽음.

## 2. Label 정책

### 허용
- namespace, app, container, pod (조심).
- level (info/warn/error).
- env, region.

### 금지
- user_id, request_id, trace_id.
- timestamp, duration.
- IP, URL path.

### 대안
금지 label은 **본문에 두기**. 본문은 인덱싱 안 함. LogQL parse로 query.

```logql
{app="myapp"} | json | user_id="alice"
```

## 3. Cost 모델 (Loki Cloud)

- Active stream 수.
- Ingest 데이터량 (GB).
- Query 빈도 + 범위.

Stream 폭발 → 비용 폭발.

## 4. 압축 + Cardinality

데이터 압축률은 본문 다양성에 영향. 같은 stream의 비슷한 본문은 압축 잘 됨.

## 5. Retention

```yaml
limits_config:
  retention_period: 30d
  
schema_config:
  configs:
    - from: 2024-01-01
      ...
      schema: v13
```

30일 hot, 그 이전 cold. Compactor가 삭제.

## 6. Cost 줄이기 5 방법

1. **Label 줄이기**: 진짜 필요한 label만.
2. **Log level filter**: DEBUG는 dev만, prod는 INFO 이상.
3. **Sampling**: 똑같은 메시지 N건당 1건.
4. **Retention 짧게**: 7d hot + 90d cold.
5. **Structured log**: JSON으로 parsing 효율.

## 7. 사례

- 10명 회사: log 1TB/month, Loki 비용 $50.
- 1000명 회사 잘못 설계: 1PB/month, $50,000.

빅테크 SLO: 사용자당 로그 비용 < $X.

## 8. Sampling 패턴

### Application level
```python
if random() < 0.1:
    log.info("...")
```

### Promtail processing
```yaml
pipeline_stages:
  - sampling:
      rate: 10  # 10%
```

### LogQL drop
이미 ingest 후엔 비용. ingest 전에 줄여야.

## 9. 모니터링 (Loki 자체)

```promql
sum by (tenant) (loki_distributor_lines_received_total)
sum by (tenant) (loki_distributor_bytes_received_total)
sum by (tenant) (loki_ingester_streams)
```

stream 수 spike 알람.

## 10. 결론

Loki cost = label 관리. 잘하면 ELK 대비 1/10, 잘못하면 ELK 못지않게 비싸짐. 시니어의 책임.
