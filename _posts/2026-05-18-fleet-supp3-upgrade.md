---
layout: post
title: "Fleet 보충 3: Cluster Upgrade Strategies"
date: 2026-05-18 23:30:00 +0900
categories: fleet upgrade supplement
---

## Strategy
1. Canary: 1 cluster 먼저.
2. Wave: tier별 (dev → stage → prod).
3. Blue-Green: 신규 cluster 띄움 → traffic 이전.

## Risk
- API 변경 (deprecated).
- CNI/CSI 호환.
- application breaking.

## Tooling
- kubent: deprecated API 검사.
- CAPI upgrade plan.

## 결론
upgrade가 cluster 운영의 가장 큰 위험.
