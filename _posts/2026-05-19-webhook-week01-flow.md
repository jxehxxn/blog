---
layout: post
title: "Webhook Week 1: Admission Flow"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook senior-series
---

## Flow
```
Request → Authn → Authz → Mutating webhook → Validating webhook → etcd
```

## Mutating
요청 수정 (patch).

## Validating
검증, reject.

## Webhook configuration
ValidatingWebhookConfiguration / MutatingWebhookConfiguration CR.

## 자가평가
### Q1. 순서? **mutating → validating**. 정답 1.
### Q2. Mutating? **요청 patch**. 정답 1.
### Q3. Validating? **검증 reject**. 정답 1.
### Q4. config CR? **MutatingWebhookConfiguration / ValidatingWebhookConfiguration**. 정답 1.
### Q5. etcd 위치? **마지막**. 정답 1.
