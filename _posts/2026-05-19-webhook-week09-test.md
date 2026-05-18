---
layout: post
title: "Webhook Week 9: Test"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook senior-series
---

## Unit
```go
func TestValidate(t *testing.T) {
    v := DeploymentValidator{}
    _, err := v.ValidateCreate(ctx, &appsv1.Deployment{...})
    require.Error(t, err)
}
```

## envtest
controller-runtime의 envtest. real API server (etcd 포함) local.

## e2e
kind cluster에 webhook 배포 후 시나리오.

## 자가평가
### Q1. unit? **interface 직접 호출**. 정답 1.
### Q2. envtest? **local API server**. 정답 1.
### Q3. e2e? **kind + webhook 배포**. 정답 1.
### Q4. CI? **GitHub Actions**. 정답 1.
### Q5. coverage? **>80%**. 정답 1.
