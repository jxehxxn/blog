---
layout: post
title: "Cost Week 7: Pod 자원 관리"
date: 2026-05-19 01:00:00 +0900
categories: cost senior-series
---

## Request vs Limit

- Request: scheduling 기준 + 보장.
- Limit: 사용 한도.

Request 너무 높으면 over-provisioning. 너무 낮으면 eviction.

## QoS Class
- Guaranteed: request == limit.
- Burstable: request < limit.
- BestEffort: 둘 다 없음.

## 권장
- request: p95 사용량.
- limit: request × 2~3.

## 자가평가
### Q1. Request? **scheduling + 보장**. 정답 1.
### Q2. Guaranteed QoS? **request == limit**. 정답 1.
### Q3. 권장 request? **p95 사용량**. 정답 1.
### Q4. limit 너무 낮음? **OOMKilled**. 정답 1.
### Q5. BestEffort 위험? **eviction 1순위**. 정답 1.
