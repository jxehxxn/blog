---
layout: post
title: "CRD Week 3: Types + Marker"
date: 2026-05-19 06:00:00 +0900
categories: crd kubebuilder senior-series
---

## Marker
```go
type MyDatabaseSpec struct {
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:Minimum=1
    Replicas int32 `json:"replicas"`

    // +kubebuilder:validation:Enum=mysql;postgres
    Engine string `json:"engine"`
}

// +kubebuilder:resource:shortName=mydb
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
type MyDatabase struct { ... }
```

## Marker 종류
- validation: Required, Minimum, Enum.
- printcolumn: kubectl get 시 컬럼.
- shortName: kubectl aliases.
- subresource: status/scale.

## 자가평가
### Q1. validation? **OpenAPI schema 생성**. 정답 1.
### Q2. subresource:status? **status update 별도 권한**. 정답 1.
### Q3. printcolumn? **kubectl get 표시**. 정답 1.
### Q4. shortName? **alias**. 정답 1.
### Q5. controller-gen? **marker 처리**. 정답 1.
