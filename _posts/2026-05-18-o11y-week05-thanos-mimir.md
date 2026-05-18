---
layout: post
title: "O11y Week 5: Thanos / Mimir — Long-term + HA Prometheus"
date: 2026-05-18 18:00:00 +0900
categories: observability prometheus thanos mimir platform senior-series
---

## 학습 목표

- 단일 Prometheus의 3대 한계.
- Thanos 아키텍처 (sidecar, store, query, compactor, ruler).
- Mimir의 다른 접근 (suite micro service).
- 선택 기준.

## 1. 단일 Prometheus 한계

1. **저장 한계**: local TSDB, 보통 15일.
2. **HA 부재**: 1대 죽으면 메트릭 손실.
3. **Cluster federated 어려움**: 여러 클러스터의 metric 통합 조회.

## 2. Thanos — Prometheus 위에 올리는 LEGO

```
[ Prometheus + thanos sidecar ] → upload to S3
       ↓
[ thanos store gateway ] ← S3
       ↓
[ thanos query ] ← 여러 store + sidecar 통합
       ↓
[ Grafana ] → query
```

### 컴포넌트
- **Sidecar**: Prometheus 옆 Pod. 2시간마다 TSDB block을 S3 업로드.
- **Store Gateway**: S3 데이터를 query interface로 제공.
- **Query**: sidecar + store + 다른 query를 통합 (PromQL 호환).
- **Compactor**: S3 block 압축, downsample (5m, 1h resolution).
- **Ruler**: rule을 별도로 평가, 결과를 S3에 저장.

### Storage Spec
S3, GCS, Azure Blob, OCI. 사실상 무한 보관.

빅테크 전형: 30일 raw + 1년 5m downsample + 5년 1h downsample.

## 3. Mimir — Grafana Labs

Cortex fork. micro-service 아키텍처.

### 컴포넌트
- distributor, ingester, querier, query-frontend, store-gateway, compactor, ruler.

각각 별도 Deployment, HPA, scale.

장점: scale 자유로움, multi-tenancy 강력.
단점: 컴포넌트 많아 운영 복잡.

## 4. Thanos vs Mimir

| 항목 | Thanos | Mimir |
|------|--------|-------|
| 아키텍처 | Prometheus + add-on | 독립 microservice suite |
| 운영 복잡도 | 중 | 중상 |
| Multi-tenancy | 기본 약함 | 강함 |
| 빅테크 | 광범위 | 증가 |
| 학습 곡선 | 비교적 완만 | 가파름 |

## 5. HA Pair Prometheus

같은 데이터 2벌 (replica). Thanos query가 dedup.

```yaml
# Prometheus
spec:
  replicas: 2
  externalLabels:
    cluster: prod
    replica: $(POD_NAME)
```

```yaml
# Thanos query
args:
  - "--query.replica-label=replica"
```

같은 metric의 두 replica 결과를 자동 dedup.

## 6. Downsampling

5분/1시간 단위로 미리 압축. 장기 보관 시 디스크/조회 비용 절감.

```bash
thanos compact \
  --retention.resolution-raw=30d \
  --retention.resolution-5m=180d \
  --retention.resolution-1h=5y
```

UI에서 query 시 thanos query가 적절한 resolution 자동 선택.

## 7. Federation의 차이

전통 federation: 한 Prometheus가 다른 Prometheus를 scrape. 작은 규모만.

Thanos/Mimir: 모든 데이터가 object storage. 통합 query 자연스러움.

## 8. Multi-cluster Pattern

각 K8s 클러스터에 Prometheus + sidecar. 모두 같은 S3 bucket에. 중앙 thanos query가 모든 데이터 조회.

```
[ cluster-1 ] → S3
[ cluster-2 ] → S3   ← thanos query → Grafana
[ cluster-3 ] → S3
```

50 클러스터도 통합 dashboard.

## 9. 운영 함정 5선

1. compactor 단일 인스턴스 (다중 시 데이터 손상).
2. S3 lifecycle 미설정 → 비용 폭증.
3. dedup label 미설정 → 중복 결과.
4. retention 너무 길게 + compaction 안 됨 → 조회 느림.
5. ruler 분리 안 함 → Prometheus 부하 가중.

## 10. 빅테크 사례

### Spotify
Thanos 표준. 100+ 클러스터 통합.

### Grafana Labs
자체 Mimir 운영, SaaS도 제공.

### Robinhood
Cortex (Mimir 전신) 운영.

## 11. 실습

```bash
# 1. kube-prometheus-stack의 thanos sidecar 활성화
# 2. MinIO를 S3 대체로 띄움
# 3. thanos query + store gateway 배포
# 4. Grafana datasource를 thanos query로
# 5. 의도적으로 Prometheus 1대 죽이고 dedup 동작 확인
```

## 12. 자가평가 퀴즈

### Q1. 단일 Prometheus 한계?
1. **저장/HA/federation 모두 약함**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q2. Thanos sidecar의 역할?
1. **TSDB block을 S3 업로드**
2. UI
3. scrape
4. 무관

**정답: 1.**

### Q3. Compactor 다중 시 위험?
1. **데이터 손상**
2. 안전
3. 빠름
4. 무관

**정답: 1.**

### Q4. Downsampling 가치?
1. **장기 보관 시 디스크/조회 비용 절감**
2. UI
3. 무관
4. 빠른 scrape

**정답: 1.**

### Q5. Multi-cluster federation 권장 방식?
1. **각 cluster sidecar → 공통 S3 → 중앙 query**
2. 전통 federation
3. UI
4. 무관

**정답: 1.**

## 13. 다음 주차

[Week 6: Loki]에서 log 적재를 다룹니다.
