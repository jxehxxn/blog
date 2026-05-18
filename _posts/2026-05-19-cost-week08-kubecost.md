---
layout: post
title: "Cost Week 8: Kubecost"
date: 2026-05-19 01:00:00 +0900
categories: cost kubecost senior-series
---

## Kubecost

K8s 비용 가시화 표준.

```bash
helm install kubecost cost-analyzer/cost-analyzer
```

## 기능
- Pod별 비용 (CPU/메모리/storage).
- Namespace allocation.
- 효율성 metric (request vs 사용).
- 권장 sizing.

## OpenCost
Kubecost의 OSS core. CNCF.

## Multi-cluster
중앙 dashboard로 통합 view.

## 자가평가
### Q1. Kubecost 가치? **K8s 비용 가시화**. 정답 1.
### Q2. OpenCost? **OSS core, CNCF**. 정답 1.
### Q3. Pod별 비용? **자동 계산**. 정답 1.
### Q4. 권장 sizing? **VPA-like 추천**. 정답 1.
### Q5. Multi-cluster? **중앙 view**. 정답 1.
