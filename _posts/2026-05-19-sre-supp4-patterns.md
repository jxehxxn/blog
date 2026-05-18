---
layout: post
title: "SRE 보충 4: Reliability Patterns Catalog"
date: 2026-05-19 07:00:00 +0900
categories: sre patterns supplement
---

## Catalog
1. Retry + jitter.
2. Circuit breaker (Hystrix style).
3. Timeout (always).
4. Bulkhead (자원 격리).
5. Rate limit (token bucket).
6. Cache (TTL + eviction).
7. Hedged request (Google paper).
8. Graceful degradation (fallback).
9. Load shedding (queue 한도).
10. Backpressure.

## 결론
distributed system의 표준 도구 상자.
