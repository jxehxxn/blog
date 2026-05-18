---
layout: post
title: "O11y 보충 1: Prometheus HA + Federation 깊게"
date: 2026-05-18 18:00:00 +0900
categories: observability prometheus platform senior-series supplement
---

5주차에서 짧게 다룬 Thanos를 운영 관점으로 더 풉니다.

## 1. HA Pair

```
[ Prometheus replica-0 ] ────┐
                              ├─→ [Thanos query (dedup)] → Grafana
[ Prometheus replica-1 ] ────┘
```

두 replica가 같은 target scrape. external label `replica=0/1`. Thanos query에서 `--query.replica-label=replica`.

장점: 1대 죽어도 무중단.
단점: scrape 부하 2배.

## 2. Sharding (Functional)

같은 cluster 안의 target을 shard:

```
[Prometheus shard-1] scrape job=app-A
[Prometheus shard-2] scrape job=app-B
```

각 shard가 자기 target만. 큰 클러스터의 scrape 부하 분산.

Prometheus Operator의 `shards` field.

## 3. Sharding (Hash-based)

같은 job을 hash로 분산:

```yaml
relabel_configs:
  - source_labels: [__address__]
    modulus: 4
    target_label: __tmp_hash
    action: hashmod
  - source_labels: [__tmp_hash]
    regex: "1"
    action: keep
```

shard-N이 hash mod N == N 인 target만.

## 4. Remote Write

Prometheus → 원격 storage push:

```yaml
remote_write:
  - url: http://thanos-receive:19291/api/v1/receive
    queue_config:
      max_samples_per_send: 5000
      capacity: 250000
      max_shards: 200
```

Thanos receive / Mimir / Cortex 등.

장점: HA receive cluster가 storage 책임. Prometheus는 가벼움.

## 5. Federation (전통, 권장 안 함)

```yaml
- job_name: federate
  honor_labels: true
  metrics_path: /federate
  params:
    'match[]':
      - '{job=~".+"}'
  static_configs:
    - targets: ['child-prometheus:9090']
```

child Prometheus의 모든 metric을 parent로 가져옴. 작은 규모만 가능. 빅테크에서는 Thanos/Mimir.

## 6. Multi-cluster Topology

### Hub-and-spoke
- 각 cluster Prometheus + Thanos sidecar.
- 모두 같은 S3.
- 중앙 Thanos query.

### Distributed
- 각 region에 Thanos query.
- 중앙 query가 region query들을 query.

빅테크 두 패턴 모두 사용.

## 7. Long-term retention 정책

- Raw 15~30d.
- 5m downsample 90~180d.
- 1h downsample 1~5y.

Compactor가 downsample.

## 8. Cost 모델

비용 주요인:
- S3 storage (저렴).
- S3 API (compactor get/put 많음).
- Compute (querier, store gateway).

빅테크 패턴: Reserved S3 bucket + 효율 query 캐싱.

## 9. Caching

- store-gateway: in-memory cache + memcached.
- query-frontend: query 결과 cache.

수십 GB memcached가 표준.

## 10. 결론

Thanos는 LEGO 처럼 운영 패턴에 맞게 조립. HA + sharding + remote write + downsample + cache를 적절히 조합해야 진짜 빅테크 scale.
