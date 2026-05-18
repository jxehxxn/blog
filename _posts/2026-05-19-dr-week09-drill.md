---
layout: post
title: "DR Week 9: DR Drill"
date: 2026-05-19 00:00:00 +0900
categories: dr drill senior-series
---

## Drill 종류
- **Tabletop**: 회의실, 시나리오 토론.
- **Restore drill**: backup → 새 cluster 복원 (자원 사용).
- **Game day**: 실 production에 부분 chaos 주입.
- **Full failover**: production region 전체 standby로.

## 빈도
- Tabletop: 분기.
- Restore: 분기.
- Game day: 월.
- Full failover: 연 1회.

## 측정
- 실제 RTO vs 목표.
- 실제 RPO vs 목표.
- gap 분석 + 개선 액션.

## 자가평가
### Q1. Tabletop? **회의실 시나리오**. 정답 1.
### Q2. Full failover? **standby로 전체 이전**. 정답 1.
### Q3. 빈도? **tabletop 분기, full 연 1**. 정답 1.
### Q4. 측정? **실제 vs 목표 RTO/RPO**. 정답 1.
### Q5. 결과? **gap 액션**. 정답 1.
