---
layout: post
title: "CRD Week 5: Owner Reference + Finalizer"
date: 2026-05-19 06:00:00 +0900
categories: crd controller senior-series
---

## OwnerReference
부모 삭제 시 자식 cascading delete.

```go
ctrl.SetControllerReference(parent, child, r.Scheme)
```

## Finalizer
삭제 직전 외부 자원 정리.

```go
if !db.DeletionTimestamp.IsZero() {
    if containsString(db.Finalizers, finalizerName) {
        cleanupExternal(&db)
        db.Finalizers = removeString(db.Finalizers, finalizerName)
        r.Update(ctx, &db)
    }
    return ctrl.Result{}, nil
}
if !containsString(db.Finalizers, finalizerName) {
    db.Finalizers = append(db.Finalizers, finalizerName)
    r.Update(ctx, &db)
}
```

## 자가평가
### Q1. OwnerRef? **cascading delete**. 정답 1.
### Q2. Finalizer? **삭제 전 정리**. 정답 1.
### Q3. DeletionTimestamp? **deletion 시작 시각**. 정답 1.
### Q4. finalizer 누락? **외부 자원 leak**. 정답 1.
### Q5. add 위치? **reconcile 초입**. 정답 1.
