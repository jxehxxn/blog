---
layout: post
title: "Interview Week 6: Design — Multi-region API"
date: 2026-05-19 08:00:00 +0900
categories: interview design senior-series
---

## Prompt
"Design API serving 100M users globally."

## 답
1. **GLB**: AWS Global Accelerator / GCP GLB.
2. **Region**: us-east-1, eu-west-1, ap-northeast-1.
3. **Compute**: K8s per region.
4. **DB**: PostgreSQL with cross-region replication (RDS Aurora Global).
5. **Cache**: Redis per region.
6. **CDN**: CloudFront.
7. **DNS**: latency-based routing.
8. **Failure**: region failover (RTO 5분).

## 자가평가
### Q1. GLB 가치? **closest region 자동**. 정답 1.
### Q2. DB? **cross-region replication**. 정답 1.
### Q3. CDN? **edge cache**. 정답 1.
### Q4. failover? **DNS + GLB health check**. 정답 1.
### Q5. trade-off? **strong consistency vs latency**. 정답 1.
