---
layout: post
title: "ArgoCD Perf Week 10: K8s API Server Impact"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance senior-series
---

## 부담
ArgoCD가 apply/get/watch 폭주 → API server CPU/메모리.

## APF
ArgoCD를 별도 priority level.

```yaml
spec:
  priorityLevelConfiguration: { name: workload-high }
  rules:
    - subjects:
        - { kind: ServiceAccount, serviceAccount: { name: argocd-application-controller, namespace: argocd } }
```

## etcd 영향
큰 app은 큰 자원 → etcd disk.

## 자가평가
### Q1. ArgoCD가 API server에? **apply/get/watch 폭주**. 정답 1.
### Q2. APF? **별도 priority**. 정답 1.
### Q3. etcd? **큰 자원 → disk 부담**. 정답 1.
### Q4. 모니터링? **apiserver_request_total**. 정답 1.
### Q5. fail-open? **ArgoCD 정지 시 cluster 영향 X**. 정답 1.
