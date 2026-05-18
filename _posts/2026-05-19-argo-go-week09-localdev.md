---
layout: post
title: "ArgoCD Go Week 9: Local Dev Environment"
date: 2026-05-19 04:00:00 +0900
extends: argocd go contributor senior-series
categories: argocd go contributor senior-series
---

## kind
```bash
kind create cluster
make install
make codegen
make controller-image
```

## 디버그
goland/vscode debugger attach.

## hot reload
나가서 다시 빌드 + restart.

## 자가평가
### Q1. local k8s? **kind**. 정답 1.
### Q2. codegen? **CRD/deepcopy 재생성**. 정답 1.
### Q3. debug? **dlv attach**. 정답 1.
### Q4. UI dev? **yarn dev (UI server에서)**. 정답 1.
### Q5. testing? **make test**. 정답 1.
