---
layout: post
title: "Webhook Week 10: Monitoring"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook senior-series
---

## Metrics
```promql
apiserver_admission_webhook_admission_duration_seconds
apiserver_admission_webhook_rejection_count_total
```

## SLO
- p99 latency < 500ms.
- error rate < 1%.

## Alert
- latency spike.
- 5xx 폭증.

## 자가평가
### Q1. p99 SLO? **500ms**. 정답 1.
### Q2. metric 출처? **K8s API server**. 정답 1.
### Q3. error rate 알람? **5xx**. 정답 1.
### Q4. webhook 자체 logs? **klog + Stackdriver/CloudWatch/Loki**. 정답 1.
### Q5. SLI rejection? **rejection_count_total**. 정답 1.
