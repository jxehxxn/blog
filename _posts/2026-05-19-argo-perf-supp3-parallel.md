---
layout: post
title: "ArgoCD Perf 보충 3: Parallel Sync"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance supplement
---

## operationProcessors
동시 sync 한도. 기본 10.

## tuning
- 100~500 app: 10~25.
- 1000+: 50.
- 5000+: 100.

## 위험
너무 크면 K8s API server 마비.

## 결론
metric 기반 점진 증가.
