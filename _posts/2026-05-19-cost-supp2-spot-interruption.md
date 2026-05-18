---
layout: post
title: "Cost 보충 2: Spot Interruption Handling"
date: 2026-05-19 01:00:00 +0900
categories: cost spot supplement
---

## AWS Node Termination Handler
2분 알람 → Pod drain.

## Karpenter 통합
Karpenter는 NTH 기능 내장.

## Application
- graceful shutdown (terminationGracePeriodSeconds).
- PodDisruptionBudget.

## 결론
Spot은 prod도 가능, 잘 설계 시.
