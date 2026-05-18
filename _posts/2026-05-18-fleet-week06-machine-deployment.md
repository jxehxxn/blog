---
layout: post
title: "Fleet Week 6: MachineDeployment + MachineSet"
date: 2026-05-18 23:30:00 +0900
categories: cluster-api fleet platform senior-series
---

## 비유 — Deployment ↔ ReplicaSet ↔ Pod
K8s의 Deployment → ReplicaSet → Pod.
CAPI의 MachineDeployment → MachineSet → Machine.

## MachineDeployment
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
spec:
  clusterName: team-a
  replicas: 5
  template:
    spec:
      version: v1.29.0
      bootstrap: { configRef: { ... } }
      infrastructureRef: { ... }
```

worker 노드 그룹.

## MachinePool
auto-scaling 그룹 wrap. ASG/MIG/VMSS와 매핑.

## 자가평가
### Q1. MD vs MS vs Machine? **계층 (Deployment 패턴)**. 정답 1.
### Q2. MachinePool? **cloud auto-scaling 그룹 wrap**. 정답 1.
### Q3. upgrade? **rolling Machine 교체**. 정답 1.
### Q4. worker scale? **replicas 변경**. 정답 1.
### Q5. version 변경? **rolling upgrade trigger**. 정답 1.

## 다음
Week 7.
