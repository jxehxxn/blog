---
layout: post
title: "CRD Week 9: Multi-version CRD + Conversion"
date: 2026-05-19 06:00:00 +0900
categories: crd controller senior-series
---

## Multi-version

```yaml
versions:
  - { name: v1alpha1, served: true, storage: false }
  - { name: v1beta1, served: true, storage: true }
  - { name: v1, served: true, storage: false }
```

## Conversion webhook
v1alpha1 ↔ v1beta1 ↔ v1.

## Hub-Spoke
hub version (저장 형식) ↔ spoke versions.

## 자가평가
### Q1. storage? **etcd에 저장되는 단일 version**. 정답 1.
### Q2. served? **API server가 serve**. 정답 1.
### Q3. conversion? **version 간 변환 webhook**. 정답 1.
### Q4. hub-spoke? **변환 toplogy**. 정답 1.
### Q5. backward compat? **conversion 핵심**. 정답 1.
