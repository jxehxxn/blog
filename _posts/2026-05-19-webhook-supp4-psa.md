---
layout: post
title: "Webhook 보충 4: PSA 내장 vs Custom"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook psa supplement
---

## PSA (Pod Security Admission)
K8s 내장. baseline/restricted. Custom webhook 부담 0.

## Custom webhook 사용 시점
- PSA 표준 외 정책.
- 사내 도메인 검증.

## 권장
- 표준 → PSA + Kyverno/Gatekeeper.
- 특수 → Custom.

## 결론
대부분 Custom 필요 없음.
