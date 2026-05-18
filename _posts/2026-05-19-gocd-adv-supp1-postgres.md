---
layout: post
title: "GoCD 심화 보충 1: PostgreSQL Backend 깊게"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd postgresql supplement
---

## DB Schema

GoCD가 PostgreSQL에 생성하는 주요 table:
- `pipelines`
- `stages`
- `builds`
- `materials`
- `modifications`
- `agents`
- `users`

수만 pipeline run = 수억 row.

## HA

- Streaming replication (sync/async).
- Patroni operator (K8s).
- pgBackRest backup.

## Performance

- shared_buffers: RAM 25%.
- work_mem: query별.
- max_connections: 200~500.
- autovacuum 적극.

## Index

자주 query 되는 column에 index:
```sql
CREATE INDEX idx_pipeline_counter ON pipelines (pipeline_id, counter DESC);
```

## Migration

GoCD upgrade가 DB schema 변경 → 자동 migration. backup 필수.

## Monitoring

```promql
pg_stat_statements_max_time
pg_database_size_bytes
pg_stat_replication_lag
```

## 결론
큰 GoCD = 큰 PostgreSQL. DBA 협업.
