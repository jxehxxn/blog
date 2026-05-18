---
layout: post
title: "Webhook Week 11: Troubleshooting"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook senior-series
---

## "Webhook timeout"
- webhook Pod 자원 부족.
- 외부 호출 latency.
- timeoutSeconds 너무 짧음.

## "CABundle mismatch"
- cert-manager cainjector 동작 확인.

## "Pod 생성 실패"
- failurePolicy Fail 인데 webhook down.
- API server 마비 위험.

## 자가평가
### Q1. timeout 흔한 원인? **webhook Pod 부담**. 정답 1.
### Q2. CABundle? **cainjector**. 정답 1.
### Q3. Pod 생성 실패? **failurePolicy Fail + webhook down**. 정답 1.
### Q4. kube-system exclude? **admission self-protection**. 정답 1.
### Q5. failure recover? **failurePolicy 임시 Ignore**. 정답 1.
