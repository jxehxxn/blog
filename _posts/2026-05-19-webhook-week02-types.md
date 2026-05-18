---
layout: post
title: "Webhook Week 2: Validating vs Mutating"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook senior-series
---

## AdmissionReview
HTTP POST JSON. 같은 형식.

## Response
- Validating: `allowed: bool, status: ...`.
- Mutating: + `patch: base64(JSON patch)`.

## 사용 시점
- 단순 검증 → Validating.
- 자동 보완 → Mutating.

## 자가평가
### Q1. AdmissionReview 형식? **JSON 표준**. 정답 1.
### Q2. Mutating 응답 추가? **patch**. 정답 1.
### Q3. patch 형식? **JSON patch (RFC 6902)**. 정답 1.
### Q4. 단순 검증? **Validating**. 정답 1.
### Q5. 자동 default? **Mutating**. 정답 1.
