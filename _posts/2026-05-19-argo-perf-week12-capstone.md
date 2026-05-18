---
layout: post
title: "ArgoCD Perf Week 12: Capstone"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance capstone
---

## 시나리오
10,000 app + 50 cluster ArgoCD.

## 산출물
- Controller shard 10.
- repo-server HPA 5~30.
- Redis HA.
- APF 설정.
- monitoring SLO + 알람.

## 결과
reconcile p95 < 30s, OOM 0, K8s API 정상.
