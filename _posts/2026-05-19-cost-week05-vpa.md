---
layout: post
title: "Cost Week 5: VPA (Vertical Pod Autoscaler)"
date: 2026-05-19 01:00:00 +0900
categories: cost vpa senior-series
---

## VPA

Pod의 resource request 자동 조정. 사용량 기반.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
spec:
  targetRef: { apiVersion: apps/v1, kind: Deployment, name: myapp }
  updatePolicy:
    updateMode: Auto    # Off / Initial / Recreate / Auto
```

## In-place resize (v1.27+ alpha)
Pod restart 없이 자원 변경. 게임체인저.

## HPA와 충돌
HPA가 replicas 조정, VPA가 자원 조정. metric (CPU) 같이 보면 충돌. Auto 대신 Initial 모드 권장.

## 자가평가
### Q1. VPA? **resource request 자동 조정**. 정답 1.
### Q2. updateMode? **Off/Initial/Recreate/Auto**. 정답 1.
### Q3. In-place resize? **restart 없이 자원 변경**. 정답 1.
### Q4. HPA와 충돌? **같은 metric 시**. 정답 1.
### Q5. 권장? **Initial mode**. 정답 1.
