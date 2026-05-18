---
layout: post
title: "Cost Week 2: Cluster Autoscaler"
date: 2026-05-19 01:00:00 +0900
categories: cost autoscaler senior-series
---

## CA 동작
- Pending Pod → 노드 추가.
- 사용량 낮은 노드 → 제거.

## 한계
- ASG/MIG 기반. node group 별로 운영.
- spec이 다른 instance 동시 사용 어려움.
- 5~10분 scale-up latency.

## Karpenter가 해결 (다음 주차).

## 자가평가
### Q1. CA trigger? **Pending Pod**. 정답 1.
### Q2. 한계? **node group 의존**. 정답 1.
### Q3. Latency? **5~10분**. 정답 1.
### Q4. node group? **ASG/MIG/VMSS**. 정답 1.
### Q5. Karpenter 차이? **node group 없이 직접**. 정답 1.
