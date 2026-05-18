---
layout: post
title: "ArgoCD Perf Week 3: Repo-server Scaling"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance senior-series
---

## repo-server
git clone + helm/kustomize render. CPU bound.

## 스케일
```yaml
repo-server:
  replicas: 10
  autoscale:
    enabled: true
    minReplicas: 5
    maxReplicas: 30
```

## Cache
- repo cache: render 결과 cache. TTL.
- git cache.

## 큰 repo 최적
- shallow clone.
- partial clone (--filter=blob:none).
- 사내 git mirror (latency).

## 자가평가
### Q1. repo-server bound? **CPU**. 정답 1.
### Q2. autoscale? **HPA**. 정답 1.
### Q3. cache 효과? **render 반복 안 함**. 정답 1.
### Q4. shallow clone? **latency 절감**. 정답 1.
### Q5. mirror? **사내 git 근접**. 정답 1.
