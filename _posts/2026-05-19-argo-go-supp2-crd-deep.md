---
layout: post
title: "ArgoCD Go 보충 2: CRD Types 깊게"
date: 2026-05-19 04:00:00 +0900
categories: argocd go contributor supplement
---

## Marker
```go
// +kubebuilder:validation:Required
// +kubebuilder:validation:Pattern=`^[a-z]+$`
Field string
```

controller-gen이 CRD OpenAPI schema 생성.

## conversion
v1alpha1 ↔ v1beta1 ↔ v1. webhook.

## 결론
CRD evolution이 ArgoCD의 backward compat 핵심.
