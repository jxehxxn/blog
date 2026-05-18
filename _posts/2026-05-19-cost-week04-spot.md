---
layout: post
title: "Cost Week 4: Spot Instance"
date: 2026-05-19 01:00:00 +0900
categories: cost spot senior-series
---

## Spot
on-demand 대비 70~90% 할인. 2분 사전 알람 후 종료 가능.

## 적합
- Stateless workload.
- Batch.
- Fault-tolerant.

## 부적합
- StatefulSet 그대로 (자체 HA 없음).
- 짧은 실행 + 재시작 비용 큰 것.

## 운영
- AWS Node Termination Handler.
- Graceful shutdown.
- Diversify (여러 instance type).

## 자가평가
### Q1. Spot 할인? **70~90%**. 정답 1.
### Q2. 사전 알람? **2분**. 정답 1.
### Q3. 적합? **stateless, batch**. 정답 1.
### Q4. NTH? **종료 handler**. 정답 1.
### Q5. Diversify 이유? **한 type spot 부족 시 대안**. 정답 1.
