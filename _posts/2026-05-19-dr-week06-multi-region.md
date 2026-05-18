---
layout: post
title: "DR Week 6: Multi-region Pattern"
date: 2026-05-19 00:00:00 +0900
categories: dr senior-series
---

## Active-Active
양쪽 cluster 동시 서비스. RTO 0. 가장 비싼.

## Active-Passive (warm)
Standby cluster 운영 중. failover 5~30분.

## Active-Passive (cold)
Standby cluster 정지. failover 1h+.

## Backup-only
별도 region에 backup만. RTO 4h+.

## 데이터 동기화
- DB: cross-region replication (PostgreSQL streaming, RDS Aurora Global).
- 객체: S3 cross-region replication.
- K8s state: Velero scheduled backup.

## 자가평가
### Q1. Active-Active RTO? **0**. 정답 1.
### Q2. Warm vs cold? **standby 운영 중 vs 정지**. 정답 1.
### Q3. DB 동기화? **streaming replication**. 정답 1.
### Q4. S3 동기화? **cross-region replication**. 정답 1.
### Q5. Backup-only RTO? **4h+**. 정답 1.
