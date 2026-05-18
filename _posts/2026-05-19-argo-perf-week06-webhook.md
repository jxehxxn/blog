---
layout: post
title: "ArgoCD Perf Week 6: Webhook Performance"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance senior-series
---

## Webhook의 효과
git push 즉시 sync trigger. polling 지연 제거.

## 부담
push 폭주 → controller queue 폭주.

## Throttling
controller가 자체 rate limit.

## 자가평가
### Q1. Webhook 가치? **즉시 sync**. 정답 1.
### Q2. 부담? **push 폭주**. 정답 1.
### Q3. Throttling? **자체 rate limit**. 정답 1.
### Q4. 대안? **polling 간격 단축**. 정답 1.
### Q5. push 빈도 큰 repo? **각 영향 분리**. 정답 1.
