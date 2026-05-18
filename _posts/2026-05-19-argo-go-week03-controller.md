---
layout: post
title: "ArgoCD Go Week 3: Application Controller Code"
date: 2026-05-19 04:00:00 +0900
categories: argocd go contributor senior-series
---

## ApplicationController
informer + work queue 패턴.

## Reconciliation
```go
func (ctrl *ApplicationController) processAppRefreshQueueItem() {
    app := loadFromQueue()
    targetObjs := getDesired(app)
    liveObjs := getLive(app)
    diff := compare(targetObjs, liveObjs)
    if app.spec.syncPolicy.automated { sync(app, diff) }
    updateStatus(app)
}
```

## 자가평가
### Q1. 패턴? **informer + work queue**. 정답 1.
### Q2. reconcile 함수? **processAppRefreshQueueItem**. 정답 1.
### Q3. diff? **target vs live**. 정답 1.
### Q4. auto sync trigger? **syncPolicy.automated**. 정답 1.
### Q5. status? **updateStatus**. 정답 1.
