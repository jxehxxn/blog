---
layout: post
title: "ArgoCD Perf Week 1: 병목 모델"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance senior-series
---

## 병목 위치
1. application-controller reconcile loop.
2. repo-server git/render.
3. K8s API server (apply 호출).
4. redis cache.
5. UI server (gRPC).

## 크기별
- ~100 app: 단일 controller OK.
- 500~2k app: sharding.
- 5k+ app: 모든 영역 튜닝.

## 자가평가
### Q1. 첫 병목 (수백 app)? **controller**. 정답 1.
### Q2. repo-server bound? **CPU (render)**. 정답 1.
### Q3. K8s API 영향? **apply 호출 폭증**. 정답 1.
### Q4. Redis? **cache HA**. 정답 1.
### Q5. 5k+ app? **모든 영역**. 정답 1.
