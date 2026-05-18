---
layout: post
title: "Ingress Week 11: Multi-cluster L7"
date: 2026-05-19 03:00:00 +0900
categories: ingress senior-series
---

## 패턴
- DNS round-robin (간단).
- Global Load Balancer (GLB) — AWS Global Accelerator, GCP GLB.
- Service Mesh multi-cluster (Course 5).
- Submariner / Skupper (cross-cluster service).

## Failover
GLB health check → unhealthy region 자동 제외.

## 자가평가
### Q1. 가장 간단? **DNS round-robin**. 정답 1.
### Q2. GLB? **cloud provider global LB**. 정답 1.
### Q3. health check? **자동 failover**. 정답 1.
### Q4. Mesh 옵션? **east-west gateway**. 정답 1.
### Q5. cross-cluster service? **Submariner/Skupper**. 정답 1.
