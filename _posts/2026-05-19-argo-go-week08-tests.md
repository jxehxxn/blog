---
layout: post
title: "ArgoCD Go Week 8: Tests"
date: 2026-05-19 04:00:00 +0900
categories: argocd go contributor senior-series
---

## Unit
`go test ./...`.

## Integration
test/e2e/ — 실제 K8s.

## CI
GitHub Actions. PR마다.

## Mock
controller-runtime fake client.

## 자가평가
### Q1. unit? **go test ./...**. 정답 1.
### Q2. e2e? **test/e2e/**. 정답 1.
### Q3. CI? **GitHub Actions**. 정답 1.
### Q4. mock? **fake client**. 정답 1.
### Q5. coverage 목표? **>80%**. 정답 1.
