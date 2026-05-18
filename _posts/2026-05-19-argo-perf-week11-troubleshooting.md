---
layout: post
title: "ArgoCD Perf Week 11: Troubleshooting"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance senior-series
---

## 시나리오

### "Sync queue가 막힘"
- statusProcessors 증대.
- shard 추가.

### "OOM"
- memory limit 증대.
- cache TTL 줄임.

### "Git timeout"
- repo-server replica.
- git mirror 도입.

### "K8s API rate limit"
- APF 활성.
- 동시 sync 줄임.

## 자가평가
### Q1. Sync queue 막힘? **statusProcessors / shard**. 정답 1.
### Q2. OOM? **limit + cache TTL**. 정답 1.
### Q3. Git timeout? **replica + mirror**. 정답 1.
### Q4. API rate limit? **APF**. 정답 1.
### Q5. 디버깅 출발? **메트릭**. 정답 1.
