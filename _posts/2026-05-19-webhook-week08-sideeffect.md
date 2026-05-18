---
layout: post
title: "Webhook Week 8: Side Effects + Timeout"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook senior-series
---

## sideEffects
- None: 부작용 없음.
- NoneOnDryRun: dry-run 시 부작용 없음.
- Some: 부작용 있음.
- Unknown: 알 수 없음.

dry-run을 위해 None 권장.

## reinvocationPolicy
- Never: 한 번만.
- IfNeeded: 다른 webhook가 변경 시 재호출.

## 자가평가
### Q1. sideEffects None? **부작용 없음**. 정답 1.
### Q2. dry-run? **None or NoneOnDryRun**. 정답 1.
### Q3. reinvocation? **다른 webhook 변경 후 재호출**. 정답 1.
### Q4. timeout 권장? **3~5초**. 정답 1.
### Q5. 외부 호출? **side effect**. 정답 1.
