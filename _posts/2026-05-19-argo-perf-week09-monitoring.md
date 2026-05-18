---
layout: post
title: "ArgoCD Perf Week 9: Monitoring Deep"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance senior-series
---

## 핵심 메트릭
```promql
argocd_app_reconcile_bucket
argocd_app_sync_total
argocd_git_request_duration_seconds_bucket
argocd_app_info{sync_status="OutOfSync"}
argocd_kubectl_exec_pending
```

## SLO
- reconcile p95 < 30s.
- sync queue depth < 100.

## 알람
- reconcile slow.
- queue 폭주.
- OOM.

## 자가평가
### Q1. reconcile p95 SLO? **30s**. 정답 1.
### Q2. queue depth 알람? **100 초과**. 정답 1.
### Q3. git latency? **request_duration_bucket**. 정답 1.
### Q4. OutOfSync 비율? **drift 감지**. 정답 1.
### Q5. kubectl exec pending? **K8s API throttle 신호**. 정답 1.
