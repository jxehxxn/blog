---
layout: post
title: "ArgoCD Go Week 7: ApplicationSet Controller"
date: 2026-05-19 04:00:00 +0900
categories: argocd go contributor senior-series
---

## Generator interface
```go
type Generator interface {
    GenerateParams(ctx context.Context, set *ApplicationSet) ([]map[string]interface{}, error)
}
```

각 8 generator가 이 인터페이스 구현.

## 새 generator 추가
1. 새 Go file.
2. interface 구현.
3. registry 등록.

## 자가평가
### Q1. interface? **Generator**. 정답 1.
### Q2. 새 generator? **interface 구현 + register**. 정답 1.
### Q3. 8 generator? **List/Cluster/Git/Matrix/Merge/SCM/PR/Decision**. 정답 1.
### Q4. Plugin generator? **HTTP service 호출**. 정답 1.
### Q5. test? **table-driven**. 정답 1.
