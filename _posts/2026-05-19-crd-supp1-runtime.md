---
layout: post
title: "CRD 보충 1: controller-runtime Deep"
date: 2026-05-19 06:00:00 +0900
categories: crd supplement
---

## Manager
controller + webhook + metric server 통합.

## Builder
```go
ctrl.NewControllerManagedBy(mgr).
    For(&dbv1.MyDatabase{}).
    Owns(&appsv1.StatefulSet{}).
    Complete(r)
```

## Predicates
이벤트 필터 (어떤 변경 시 reconcile).

## 결론
controller-runtime이 boilerplate 90% 제거.
