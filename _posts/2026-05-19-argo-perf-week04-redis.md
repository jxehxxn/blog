---
layout: post
title: "ArgoCD Perf Week 4: Redis HA"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance redis senior-series
---

## Redis 책임
- repo render cache.
- session.
- throttling 데이터.

## HA
- redis-ha (3 instance sentinel).
- 외부 Redis cluster (대규모).

## eviction
LRU. memory 한계 도달 시 cache miss.

## monitoring
hit rate, eviction count.

## 자가평가
### Q1. Redis 책임? **cache + session**. 정답 1.
### Q2. HA? **redis-ha sentinel**. 정답 1.
### Q3. eviction policy? **LRU**. 정답 1.
### Q4. hit rate 낮음? **cache 부족 또는 TTL 짧음**. 정답 1.
### Q5. 큰 cluster? **외부 Redis**. 정답 1.
