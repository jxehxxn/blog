---
layout: post
title: "CRD Week 12: Capstone — Production Operator"
date: 2026-05-19 06:00:00 +0900
categories: crd controller capstone senior-series
---

## 시나리오
사내 MyDatabase operator. RDS + monitoring + backup 자동.

## 산출물
1. kubebuilder code.
2. CRD multi-version + conversion.
3. webhook (validate + default).
4. test (unit + envtest + e2e).
5. OperatorHub publish (또는 사내 catalog).
