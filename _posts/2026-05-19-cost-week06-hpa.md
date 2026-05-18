---
layout: post
title: "Cost Week 6: HPA (Horizontal Pod Autoscaler)"
date: 2026-05-19 01:00:00 +0900
categories: cost hpa senior-series
---

## HPA

replicas 조정. CPU/메모리/custom metric.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef: { kind: Deployment, name: myapp }
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

## KEDA
event-driven autoscaler. Kafka queue length, SQS depth 등.

## 자가평가
### Q1. HPA 조정? **replicas**. 정답 1.
### Q2. metric? **CPU/memory/custom**. 정답 1.
### Q3. KEDA? **event-driven**. 정답 1.
### Q4. minReplicas? **scale-in 한도**. 정답 1.
### Q5. CPU 70%? **target utilization**. 정답 1.
