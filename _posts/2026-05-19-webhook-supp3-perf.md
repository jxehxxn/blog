---
layout: post
title: "Webhook 보충 3: Performance"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook performance supplement
---

## 부담
모든 매칭 요청에 webhook latency. p99 500ms 목표.

## 최적화
- in-memory cache (반복 매칭).
- HPA로 webhook Pod scale.
- 외부 호출 최소.

## scale
큰 cluster: webhook Pod 5~10 replica.

## 결론
admission webhook이 cluster 전체 latency에 직결.
