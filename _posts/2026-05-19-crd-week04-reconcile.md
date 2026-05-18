---
layout: post
title: "CRD Week 4: Reconcile Loop"
date: 2026-05-19 06:00:00 +0900
categories: crd controller senior-series
---

## Reconcile

```go
func (r *MyDatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var db dbv1.MyDatabase
    if err := r.Get(ctx, req.NamespacedName, &db); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Desired
    desired := buildStatefulSet(&db)
    
    // Actual
    var actual appsv1.StatefulSet
    err := r.Get(ctx, req.NamespacedName, &actual)
    if errors.IsNotFound(err) {
        ctrl.SetControllerReference(&db, desired, r.Scheme)
        return ctrl.Result{Requeue: true}, r.Create(ctx, desired)
    }

    // Update if differ
    if *actual.Spec.Replicas != db.Spec.Replicas {
        actual.Spec.Replicas = &db.Spec.Replicas
        return ctrl.Result{}, r.Update(ctx, &actual)
    }

    // Status update
    db.Status.ReadyReplicas = actual.Status.ReadyReplicas
    return ctrl.Result{RequeueAfter: 30 * time.Second}, r.Status().Update(ctx, &db)
}
```

## 자가평가
### Q1. reconcile 본질? **desired vs actual 차이 보정**. 정답 1.
### Q2. Get IsNotFound? **삭제됨 → 무시**. 정답 1.
### Q3. SetControllerReference? **owner ref**. 정답 1.
### Q4. RequeueAfter? **주기 reconcile**. 정답 1.
### Q5. Status update 분리? **subresource 권한**. 정답 1.
