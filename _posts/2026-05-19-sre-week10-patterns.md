---
layout: post
title: "SRE Week 10: Reliability Patterns"
date: 2026-05-19 07:00:00 +0900
categories: sre reliability senior-series
---

## Patterns
- Retry with backoff (jitter).
- Circuit breaker.
- Timeout.
- Bulkhead.
- Rate limit.
- Cache.
- Hedged request.
- Graceful degradation.

## Anti-patterns
- Infinite retry.
- No timeout.
- Cascading failure (no circuit breaker).

## 자가평가
### Q1. retry + jitter? **thundering herd 방지**. 정답 1.
### Q2. Circuit breaker? **cascading 방지**. 정답 1.
### Q3. Bulkhead? **자원 격리**. 정답 1.
### Q4. Hedged request? **여러 backend 동시 + 빠른 응답**. 정답 1.
### Q5. graceful degradation? **부분 기능 유지**. 정답 1.
