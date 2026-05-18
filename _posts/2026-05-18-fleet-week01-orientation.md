---
layout: post
title: "Fleet Week 1: Cluster를 코드로"
date: 2026-05-18 23:30:00 +0900
categories: cluster-api fleet platform senior-series
---

## 비유 — "공장 자체를 도면으로"

한 공장(cluster) 안의 기계는 IaC로. 공장 자체도 도면으로 → CAPI.

## 도구
- Cluster API: CNCF, K8s sub-project.
- Rancher Fleet: 비슷한 영역.
- OCM (Open Cluster Management): RH.
- KubeFed: 옛 federation.

## 가치
50+ cluster 운영, 신규 cluster 5분, 표준화.

## 사용 사례
- Edge (수백 cluster).
- multi-region/cloud.
- tenant cluster (vCluster + CAPI).

## 자가평가
### Q1. CAPI? **K8s 클러스터를 CRD로**. 정답 1.
### Q2. OCM? **multi-cluster orchestration**. 정답 1.
### Q3. 사용 시점? **50+ cluster**. 정답 1.
### Q4. KubeFed? **deprecated, CAPI 권장**. 정답 1.
### Q5. Fleet Rancher? **Rancher의 multi-cluster GitOps**. 정답 1.

## 다음
Week 2.
