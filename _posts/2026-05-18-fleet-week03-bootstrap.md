---
layout: post
title: "Fleet Week 3: Bootstrap (kubeadm provider)"
date: 2026-05-18 23:30:00 +0900
categories: cluster-api fleet platform senior-series
---

## Bootstrap Provider

새 노드가 cluster에 join할 때 cloud-init script 생성.

## KubeadmConfig
```yaml
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfigTemplate
metadata: { name: worker-template }
spec:
  template:
    spec:
      joinConfiguration:
        nodeRegistration: { kubeletExtraArgs: { cloud-provider: aws } }
```

## InitConfiguration vs JoinConfiguration
- Init: 첫 control plane.
- Join: 추가 노드.

## 자가평가
### Q1. Bootstrap 역할? **cloud-init script 생성**. 정답 1.
### Q2. Init vs Join? **첫 cp vs 추가 노드**. 정답 1.
### Q3. KubeadmConfigTemplate? **재사용 가능 template**. 정답 1.
### Q4. cloud-provider 옵션? **CCM 통합**. 정답 1.
### Q5. 기타 provider? **MicroK8s, Talos, K3s**. 정답 1.

## 다음
Week 4.
