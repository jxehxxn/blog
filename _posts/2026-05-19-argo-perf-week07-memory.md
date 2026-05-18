---
layout: post
title: "ArgoCD Perf Week 7: Memory Tuning"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance senior-series
---

## Memory profile

```yaml
controller:
  resources:
    requests: { memory: 4Gi }
    limits: { memory: 16Gi }
```

5k app: 16~32GB 일반.

## Cache size

repo-server cache, redis 모두 메모리 영향.

## OOM
빈번하면 limit 증대 또는 sharding.

## 자가평가
### Q1. 5k app memory? **16~32GB**. 정답 1.
### Q2. OOM 대책? **limit 증대 + shard**. 정답 1.
### Q3. Cache? **memory bound**. 정답 1.
### Q4. controller vs repo-server? **둘 다 memory 중요**. 정답 1.
### Q5. monitoring? **memory_usage_bytes**. 정답 1.
