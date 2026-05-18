---
layout: post
title: "SRE Week 5: On-call"
date: 2026-05-19 07:00:00 +0900
categories: sre oncall senior-series
---

## 원칙
- 압박 없음 (sustainable).
- 회복 가능 (8 알람/shift 한도).
- 보상.

## Schedule
1주 cycle. primary + secondary.

## PagerDuty 통합
escalation policy. 1차 5분 없으면 secondary, 다시 5분 없으면 manager.

## Runbook 필수
각 alert에 runbook URL.

## 자가평가
### Q1. 알람 한도? **8/shift**. 정답 1.
### Q2. cycle? **1주**. 정답 1.
### Q3. escalation? **primary → secondary → manager**. 정답 1.
### Q4. runbook? **각 alert 필수**. 정답 1.
### Q5. 보상? **수당/대휴**. 정답 1.
