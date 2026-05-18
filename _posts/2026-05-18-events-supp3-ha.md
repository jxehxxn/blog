---
layout: post
title: "Events 보충 3: EventBus HA + DR"
date: 2026-05-18 23:00:00 +0900
categories: argo-events supplement
---

## HA
NATS 3 replica + PVC.

## DR
- Stream snapshot.
- Disaster region에 standby cluster.
- Failover 절차 정의.

## Monitoring
- jet stream metric.
- lag, consumer pending.

## 결론
EventBus 장애 = 전체 자동화 마비. HA + DR 필수.
