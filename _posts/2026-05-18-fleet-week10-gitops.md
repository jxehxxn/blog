---
layout: post
title: "Fleet Week 10: Multi-cluster GitOps"
date: 2026-05-18 23:30:00 +0900
categories: fleet gitops platform senior-series
---

## 패턴 1: ArgoCD ApplicationSet
중앙 ArgoCD가 모든 cluster에 fan-out.

## 패턴 2: ArgoCD per cluster
각 cluster 자체 ArgoCD. 격리.

## 패턴 3: OCM ApplicationSet generator
OCM과 ArgoCD 결합.

## 패턴 4: CAPI Cluster + ArgoCD self-bootstrap
CAPI가 cluster 생성 → 그 안에 ArgoCD 자동 설치 → app sync.

## 자가평가
### Q1. 패턴 1 단점? **중앙 ArgoCD 장애 전체 영향**. 정답 1.
### Q2. 패턴 2 단점? **운영 N배**. 정답 1.
### Q3. Bootstrap pattern? **CAPI + ArgoCD self-install**. 정답 1.
### Q4. OCM generator? **ManagedCluster → Application**. 정답 1.
### Q5. 빅테크? **혼합**. 정답 1.

## 다음
Week 11.
