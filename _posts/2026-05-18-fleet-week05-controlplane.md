---
layout: post
title: "Fleet Week 5: ControlPlane Provider"
date: 2026-05-18 23:30:00 +0900
categories: cluster-api fleet platform senior-series
---

## KubeadmControlPlane

```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata: { name: team-a-cp }
spec:
  replicas: 3
  version: v1.29.0
  machineTemplate:
    infrastructureRef: { ... }
  kubeadmConfigSpec:
    clusterConfiguration: ...
```

3 replica = HA control plane.

## Upgrade

version 변경 → rolling upgrade. 1대씩 새 버전으로.

## EKS Control Plane
별도 CRD (`AWSManagedControlPlane`). AWS가 관리.

## 자가평가
### Q1. KCP 표준 replica? **3**. 정답 1.
### Q2. Upgrade 방식? **rolling**. 정답 1.
### Q3. AWS managed? **AWSManagedControlPlane**. 정답 1.
### Q4. version field? **K8s 버전**. 정답 1.
### Q5. HA 필수? **production 기준**. 정답 1.

## 다음
Week 6.
