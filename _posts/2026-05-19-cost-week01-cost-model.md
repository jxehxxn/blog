---
layout: post
title: "Cost Week 1: Cost 모델"
date: 2026-05-19 01:00:00 +0900
categories: cost senior-series
---

## 비용 구성
- Compute (EC2/GCE).
- Storage (EBS/PD).
- Network (egress, NAT).
- Managed services (RDS, ElastiCache).

빅테크 K8s: compute가 70%.

## Allocation 어려움
공유 cluster → 어느 팀의 자원? Kubecost가 해결.

## Optimization 방향
- Right-sizing.
- Spot.
- Autoscaling.
- 정리 (unused).

## 자가평가
### Q1. K8s 비용 70%? **compute**. 정답 1.
### Q2. Allocation 도구? **Kubecost**. 정답 1.
### Q3. Optimization 4? **right-size/spot/autoscale/정리**. 정답 1.
### Q4. Network 비용 흔한? **egress**. 정답 1.
### Q5. FinOps? **비용 문화**. 정답 1.
