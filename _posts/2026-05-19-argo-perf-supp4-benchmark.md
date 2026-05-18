---
layout: post
title: "ArgoCD Perf 보충 4: Benchmark Methodology"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance supplement
---

## 부하 시나리오
- N application 생성.
- 동시 sync trigger.
- 측정: latency, throughput, error rate.

## 도구
- argocd CLI script.
- K8s API direct (Go).
- Prometheus metric.

## 빅테크 reference
Intuit: 10,000 app benchmark. p95 reconcile < 1m.

## 결론
재현 가능 benchmark가 capacity planning의 기초.
