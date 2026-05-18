---
layout: post
title: "DR Week 1: RTO / RPO 정의"
date: 2026-05-19 00:00:00 +0900
categories: dr senior-series
---

## RTO (Recovery Time Objective)
사고부터 복구까지 허용 시간. SLA 결정.

## RPO (Recovery Point Objective)
허용 데이터 손실 시간. backup 빈도 결정.

## Tier
- Tier 0: RTO 0, RPO 0 (active-active).
- Tier 1: RTO 1h, RPO 5min.
- Tier 2: RTO 4h, RPO 1h.
- Tier 3: RTO 1day, RPO 1day.

## 비용 곡선
RTO/RPO 짧을수록 cost 기하급수.

## 자가평가
### Q1. RTO? **복구 허용 시간**. 정답 1.
### Q2. RPO? **데이터 손실 허용**. 정답 1.
### Q3. Tier 0? **active-active, RPO 0**. 정답 1.
### Q4. 비용? **짧을수록 기하급수**. 정답 1.
### Q5. SLA 결정? **RTO/RPO 기준**. 정답 1.
