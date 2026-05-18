---
layout: post
title: "ArgoCD Perf Week 5: ApplicationSet Scaling"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance senior-series
---

## ApplicationSet controller
generator → N Application 생성/관리.

## 큰 ApplicationSet
- Matrix가 1000+ Application → controller 부담.
- requeueAfterSeconds 짧으면 폭증.

## 튜닝
- requeueAfterSeconds 합리적 (60s+).
- ApplicationSet 분할.
- 별도 namespace로 격리.

## 자가평가
### Q1. ApplicationSet 부하? **N Application 생성**. 정답 1.
### Q2. Matrix 위험? **폭발**. 정답 1.
### Q3. requeue 권장? **60s+**. 정답 1.
### Q4. 분할? **큰 set → 작은 set 여러 개**. 정답 1.
### Q5. preserveResourcesOnDeletion? **leak 주의**. 정답 1.
