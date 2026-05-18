---
layout: post
title: "CRD Week 8: Test (envtest, e2e)"
date: 2026-05-19 06:00:00 +0900
categories: crd controller test senior-series
---

## envtest

```go
testEnv = &envtest.Environment{ CRDDirectoryPaths: []string{"config/crd/bases"} }
cfg, _ := testEnv.Start()
```

local API server (etcd 포함). 빠름.

## ginkgo + gomega
BDD style.

## e2e
kind cluster + 실제 controller 배포.

## 자가평가
### Q1. envtest? **local API server**. 정답 1.
### Q2. ginkgo? **BDD style**. 정답 1.
### Q3. e2e? **kind + controller**. 정답 1.
### Q4. CI? **GitHub Actions**. 정답 1.
### Q5. coverage 목표? **>80%**. 정답 1.
