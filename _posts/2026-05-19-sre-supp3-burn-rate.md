---
layout: post
title: "SRE 보충 3: Multi-window Multi-burn-rate"
date: 2026-05-19 07:00:00 +0900
categories: sre slo supplement
---

## Burn rate
budget 소진 속도. 1.0 = 평균.

## Multi-window
- 1h + 5m: fast burn (page).
- 6h + 1h: slow burn (warning).

## 14.4x burn (1h)
2시간 안에 budget 소진. immediate page.

## 결론
Google SRE 표준 alert 패턴.
