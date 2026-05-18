---
layout: post
title: "Webhook Week 5: failurePolicy"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook senior-series
---

## failurePolicy
- Fail: webhook 호출 실패 시 admission deny (안전).
- Ignore: 무시 (가용성).

## 권장
- critical (보안 정책): Fail.
- optional (로깅): Ignore.

## timeoutSeconds
기본 10초. 짧게 (3~5초) 권장.

## 자가평가
### Q1. Fail? **deny → 안전**. 정답 1.
### Q2. Ignore? **통과 → 가용성**. 정답 1.
### Q3. critical? **Fail**. 정답 1.
### Q4. timeout? **3~5초**. 정답 1.
### Q5. 권장? **policy에 따라**. 정답 1.
