---
layout: post
title: "CRD Week 6: Event + Status Condition"
date: 2026-05-19 06:00:00 +0900
categories: crd controller senior-series
---

## Event

```go
r.Recorder.Event(&db, "Normal", "Created", "StatefulSet created")
```

`kubectl describe`에 표시.

## Condition

```go
condition := metav1.Condition{
    Type: "Ready",
    Status: metav1.ConditionTrue,
    Reason: "AllReplicasReady",
    Message: "5/5 ready",
    LastTransitionTime: metav1.Now(),
}
meta.SetStatusCondition(&db.Status.Conditions, condition)
```

표준 condition pattern.

## 자가평가
### Q1. Event? **describe 표시**. 정답 1.
### Q2. Condition? **표준 status 표현**. 정답 1.
### Q3. meta.SetStatusCondition? **idempotent setter**. 정답 1.
### Q4. type? **Ready/Progressing/Degraded**. 정답 1.
### Q5. lastTransitionTime? **change 시각**. 정답 1.
