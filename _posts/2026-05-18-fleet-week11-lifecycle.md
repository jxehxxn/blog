---
layout: post
title: "Fleet Week 11: Cluster Lifecycle — Upgrade, Delete, DR"
date: 2026-05-18 23:30:00 +0900
categories: fleet lifecycle platform senior-series
---

## Upgrade

CAPI: `version` 필드 변경 → rolling.

전략:
- canary cluster 먼저.
- 같은 day-of-week upgrade.
- rollback plan.

## Delete

```bash
kubectl delete cluster team-a
```

CAPI가 cloud 자원 모두 정리. Workload는 사전 evict.

## Backup
etcd snapshot은 cluster 별. CAPI 자체는 stateless.

## DR
management cluster 자체 backup 필수. 손실 시 모든 workload cluster orphan.

## 자가평가
### Q1. Upgrade? **version 변경 → rolling**. 정답 1.
### Q2. Delete? **CAPI가 cloud 정리**. 정답 1.
### Q3. management cluster 손실? **모든 workload orphan**. 정답 1.
### Q4. Canary upgrade? **소수 cluster 먼저**. 정답 1.
### Q5. etcd backup? **cluster별**. 정답 1.

## 다음
Week 12.
