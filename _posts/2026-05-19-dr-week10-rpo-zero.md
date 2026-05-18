---
layout: post
title: "DR Week 10: RPO 0 — Sync Replication"
date: 2026-05-19 00:00:00 +0900
categories: dr senior-series
---

## RPO 0 = 데이터 손실 0

요구: sync replication (transaction commit이 양쪽에 적용).

## PG sync standby
`synchronous_commit = on`, `synchronous_standby_names`.

## Aurora Global
AWS RDS Aurora Global Database. RPO < 1초.

## Trade-off
- Sync: latency 증가 (양쪽 commit 대기).
- 비용: 2배 인프라.

## 적용
Tier 0만. 결제, 인증.

## 자가평가
### Q1. RPO 0 요구? **sync replication**. 정답 1.
### Q2. PG sync? **synchronous_standby_names**. 정답 1.
### Q3. Aurora? **RPO < 1초**. 정답 1.
### Q4. trade-off? **latency + cost**. 정답 1.
### Q5. 적용? **Tier 0만**. 정답 1.
