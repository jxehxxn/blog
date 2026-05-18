---
layout: post
title: "Fleet Week 9: Placement"
date: 2026-05-18 23:30:00 +0900
categories: ocm fleet platform senior-series
---

## Placement

어떤 ManagedCluster에 ManifestWork 배포할지.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchLabels: { env: prod }
  numberOfClusters: 5
  prioritizerPolicy:
    mode: Additive
    configurations:
      - scoreCoordinate: { type: BuiltIn, builtIn: ResourceAllocatableMemory }
```

cluster 5개 (prod 라벨) 선택, memory 많은 곳 우선.

## PlacementDecision

선택 결과:
```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: PlacementDecision
status:
  decisions:
    - { clusterName: cluster-1 }
    - { clusterName: cluster-2 }
```

## ManifestWork와 결합

ManifestWork를 Placement 결과 cluster 모두에 자동 생성.

## 자가평가
### Q1. Placement? **배포 cluster 선택**. 정답 1.
### Q2. numberOfClusters? **선택 한도**. 정답 1.
### Q3. prioritizerPolicy? **선택 우선순위 (cpu/mem 등)**. 정답 1.
### Q4. PlacementDecision? **선택 결과 status**. 정답 1.
### Q5. ManifestWork와 결합? **선택 cluster 모두 배포**. 정답 1.

## 다음
Week 10.
