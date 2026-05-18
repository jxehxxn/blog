---
layout: post
title: "ArgoCD Go Week 6: CRD Types"
date: 2026-05-19 04:00:00 +0900
categories: argocd go contributor senior-series
---

## pkg/apis/application/v1alpha1
- Application
- AppProject
- ApplicationSet

각 type struct.

## DeepCopy / runtime.Object
controller-gen이 자동 생성.

## OpenAPI schema
zz_generated_openapi.go.

## 자가평가
### Q1. CRD location? **pkg/apis/...**. 정답 1.
### Q2. DeepCopy 자동? **controller-gen**. 정답 1.
### Q3. 3 main type? **Application/AppProject/ApplicationSet**. 정답 1.
### Q4. OpenAPI? **zz_generated**. 정답 1.
### Q5. 새 field 추가? **type + regenerate**. 정답 1.
