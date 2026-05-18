---
layout: post
title: "DR 보충 4: Cross-cloud DR"
date: 2026-05-19 00:00:00 +0900
categories: dr supplement
---

## 동기
한 cloud 전체 장애 (실제 사례 다수).

## 도전
- 데이터 동기화.
- 네트워크 latency.
- IAM 차이.

## 도구
- Velero (cross-cloud restore).
- DB: 외부 streaming (Debezium).
- DNS failover.

## 결론
극히 어려움. Tier 0 critical만.
