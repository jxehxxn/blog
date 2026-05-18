---
layout: post
title: "SRE Week 3: Error Budget"
date: 2026-05-19 07:00:00 +0900
categories: sre slo senior-series
---

## 정의
budget = 1 - SLO. 허용 가능 실패 양.

99.9% SLO → 0.1% budget → 월 100만 요청 중 1000 fail 허용.

## 정책
- budget < 50% → 신규 feature 신중.
- budget < 25% → feature freeze.
- budget > 80% → 의도적 chaos engineering 가능.

## 의미
"safety vs velocity" trade-off를 정량으로 관리.

## 자가평가
### Q1. budget? **1 - SLO**. 정답 1.
### Q2. 99.9% SLO budget? **0.1%**. 정답 1.
### Q3. freeze 시점? **<25%**. 정답 1.
### Q4. chaos 시점? **>80% (여유)**. 정답 1.
### Q5. 본질? **safety vs velocity 정량**. 정답 1.
