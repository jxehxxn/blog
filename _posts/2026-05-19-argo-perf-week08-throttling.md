---
layout: post
title: "ArgoCD Perf Week 8: Throttling + Rate Limit"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance senior-series
---

## Operation processors
```yaml
controller:
  operationProcessors: 25       # 동시 sync 한도
  statusProcessors: 50           # 동시 reconcile
  appResyncJitter: "300s"        # 자체 jitter (debounce)
```

## API server 영향
ArgoCD가 K8s API server에 apply/get 폭주 → APF 권장.

## 자가평가
### Q1. operationProcessors? **동시 sync**. 정답 1.
### Q2. statusProcessors? **동시 reconcile**. 정답 1.
### Q3. K8s APF? **API server 폭주 방지**. 정답 1.
### Q4. jitter? **resync 분산**. 정답 1.
### Q5. throttling 부재? **K8s API 부하 폭증**. 정답 1.
