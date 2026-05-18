---
layout: post
title: "CRD 보충 4: Operator Performance Tuning"
date: 2026-05-19 06:00:00 +0900
categories: crd performance supplement
---

## Rate limit
work queue rate limit. 폭주 방지.

## Concurrency
```go
Complete(r).WithOptions(controller.Options{MaxConcurrentReconciles: 10})
```

## Resync period
긴 resync (1h)으로 부담 줄임.

## 결론
큰 자원 수 operator는 위 3 영역 튜닝.
