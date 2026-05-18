---
layout: post
title: "Fleet Week 7: Open Cluster Management (OCM)"
date: 2026-05-18 23:30:00 +0900
categories: ocm fleet platform senior-series
---

## OCM 소개

Red Hat 주도. 멀티 cluster 통합 관리. CAPI보다 다른 영역 (운영 측면).

## 컴포넌트
- Hub: 관리 cluster.
- Klusterlet: 각 managed cluster의 agent.
- ManifestWork: hub → managed cluster에 manifest 배포.
- Placement: 어떤 cluster에 배포할지 결정.

## ManagedCluster
```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata: { name: team-a-prod }
spec:
  hubAcceptsClient: true
```

## CAPI와 차이
- CAPI: cluster 자체 생성.
- OCM: 이미 있는 cluster들 운영.

병행 가능.

## 자가평가
### Q1. OCM 책임? **multi-cluster 운영 (배포 후)**. 정답 1.
### Q2. Hub/Klusterlet? **중앙/agent**. 정답 1.
### Q3. ManifestWork? **manifest 분배**. 정답 1.
### Q4. CAPI vs OCM? **생성 vs 운영, 병행 가능**. 정답 1.
### Q5. Placement? **어디 배포 결정**. 정답 1.

## 다음
Week 8.
