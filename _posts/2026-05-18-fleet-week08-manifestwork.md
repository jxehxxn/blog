---
layout: post
title: "Fleet Week 8: ManifestWork"
date: 2026-05-18 23:30:00 +0900
categories: ocm fleet platform senior-series
---

## ManifestWork

hub → managed cluster에 manifest 배포.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata: { name: deploy-nginx, namespace: team-a-prod }
spec:
  workload:
    manifests:
      - apiVersion: apps/v1
        kind: Deployment
        ...
```

namespace = managed cluster name.

## Status

각 manifest의 적용 결과 status에 보고.

## 사용
- ArgoCD ApplicationSet과 결합.
- OCM ApplicationSet generator도 있음.

## 자가평가
### Q1. ManifestWork 역할? **hub → managed manifest**. 정답 1.
### Q2. namespace = ? **managed cluster name**. 정답 1.
### Q3. status? **각 manifest 결과 보고**. 정답 1.
### Q4. ArgoCD 통합? **ApplicationSet generator**. 정답 1.
### Q5. CR 위치? **hub cluster**. 정답 1.

## 다음
Week 9.
