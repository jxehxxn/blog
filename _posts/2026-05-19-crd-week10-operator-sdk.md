---
layout: post
title: "CRD Week 10: operator-sdk 비교"
date: 2026-05-19 06:00:00 +0900
categories: crd operator-sdk senior-series
---

## operator-sdk
Red Hat. Go / Helm / Ansible based operator.

## vs kubebuilder
- kubebuilder: 더 lean, Go only.
- operator-sdk: Helm/Ansible 지원, OperatorHub 통합.

## 선택
- 빅테크 Go 표준 → kubebuilder.
- OpenShift / Helm-based → operator-sdk.

## 자가평가
### Q1. operator-sdk 출처? **Red Hat**. 정답 1.
### Q2. Helm operator? **operator-sdk만 지원**. 정답 1.
### Q3. kubebuilder 강점? **lean Go**. 정답 1.
### Q4. OpenShift? **operator-sdk 자연**. 정답 1.
### Q5. 둘 다 controller-runtime? **공통 base**. 정답 1.
