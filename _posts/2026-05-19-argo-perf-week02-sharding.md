---
layout: post
title: "ArgoCD Perf Week 2: Controller Sharding"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance senior-series
---

## Sharding 알고리즘

```yaml
controller.replicas: 5
controller.sharding.algorithm: round-robin  # 또는 legacy, consistent-hashing
```

## 동작
각 shard가 cluster 일부 관리. cluster ID hash 기반.

## consistent-hashing
shard 추가/삭제 시 minimal reassignment.

## 권장
- 5~10 cluster당 1 shard.
- 큰 cluster (1000+ Pod) 는 전용 shard.

## 자가평가
### Q1. Sharding key? **cluster**. 정답 1.
### Q2. consistent hashing? **재할당 최소**. 정답 1.
### Q3. 권장 비율? **5~10 cluster당 1 shard**. 정답 1.
### Q4. round-robin? **단순 분배**. 정답 1.
### Q5. 큰 cluster? **전용 shard**. 정답 1.
