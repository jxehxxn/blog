---
layout: post
title: "ArgoCD Go 보충 3: e2e Test"
date: 2026-05-19 04:00:00 +0900
categories: argocd go contributor supplement
---

## test/e2e
kind cluster에 실제 ArgoCD 띄움 + 시나리오.

## Fixture
`test/e2e/fixture/` 에 helper.

## 시나리오
- Application sync.
- ApplicationSet generator.
- RBAC enforce.

## 결론
PR 머지 전 e2e 강제.
