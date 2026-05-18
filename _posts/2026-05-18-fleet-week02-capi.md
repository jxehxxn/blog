---
layout: post
title: "Fleet Week 2: Cluster API 소개"
date: 2026-05-18 23:30:00 +0900
categories: cluster-api fleet platform senior-series
---

## 구조

```
[Management cluster]
   ↓ CRDs: Cluster, MachineDeployment, ...
   ↓ providers: kubeadm, CAPA(AWS), CAPG(GCP)
[Workload clusters] ← 생성됨
```

## Management Cluster

CAPI controller 운영. 다른 workload cluster 생성/관리.

## Providers
- Bootstrap: kubeadm (표준).
- Control plane: KubeadmControlPlane.
- Infra: CAPA, CAPG, CAPV (vSphere), CAPZ (Azure).

## CR 예
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata: { name: team-a }
spec:
  clusterNetwork:
    pods: { cidrBlocks: [10.244.0.0/16] }
  controlPlaneRef: { kind: KubeadmControlPlane, name: team-a-cp }
  infrastructureRef: { kind: AWSCluster, name: team-a-aws }
```

## 자가평가
### Q1. Management vs workload? **Management가 다른 cluster 관리**. 정답 1.
### Q2. Providers 3 종? **Bootstrap/ControlPlane/Infra**. 정답 1.
### Q3. CAPA? **AWS provider**. 정답 1.
### Q4. Cluster CR? **클러스터 자체 정의**. 정답 1.
### Q5. ArgoCD 통합? **Cluster CR sync**. 정답 1.

## 다음
Week 3.
