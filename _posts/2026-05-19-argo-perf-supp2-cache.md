---
layout: post
title: "ArgoCD Perf 보충 2: Cache Strategy"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance supplement
---

## Cache 종류
- repo cache (rendered manifest).
- live cache (cluster state).
- diff cache.

## TTL
- repo: 24h (또는 git revision 기준).
- live: 짧음.

## Eviction
Redis LRU.

## 효과
cache hit 99% → render 시간 거의 0.
