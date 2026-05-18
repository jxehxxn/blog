---
layout: post
title: "Webhook Week 7: Mutating Patch"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook mutating senior-series
---

## JSON Patch (RFC 6902)
```json
[
  {"op": "add", "path": "/metadata/labels/team", "value": "payments"},
  {"op": "replace", "path": "/spec/containers/0/imagePullPolicy", "value": "IfNotPresent"}
]
```

## Strategic Merge Patch (alternative)
K8s 표준. list element key 기반 merge.

## controller-runtime
CustomDefaulter interface가 obj 변경 → 자동 patch 생성.

## 자가평가
### Q1. JSON Patch RFC? **6902**. 정답 1.
### Q2. op 종류? **add/replace/remove/move/copy/test**. 정답 1.
### Q3. Strategic Merge? **K8s 표준**. 정답 1.
### Q4. controller-runtime? **obj 수정 → patch 자동**. 정답 1.
### Q5. base64 encoding? **response patch field**. 정답 1.
