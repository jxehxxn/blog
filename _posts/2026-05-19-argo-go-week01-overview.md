---
layout: post
title: "ArgoCD Go Week 1: Repo Overview"
date: 2026-05-19 04:00:00 +0900
categories: argocd go contributor senior-series
---

## Clone
```bash
git clone https://github.com/argoproj/argo-cd.git
cd argo-cd
```

## 디렉토리
- `cmd/`: binary 진입점.
- `controller/`: application controller.
- `server/`: API server.
- `reposerver/`: repo-server.
- `applicationset/`: applicationset controller.
- `pkg/apis/`: CRD types.
- `util/`: helper.

## Build
```bash
make build
```

## 자가평가
### Q1. application controller 위치? **controller/**. 정답 1.
### Q2. API server 위치? **server/**. 정답 1.
### Q3. CRD types? **pkg/apis/**. 정답 1.
### Q4. 진입점? **cmd/**. 정답 1.
### Q5. 빌드? **make build**. 정답 1.
