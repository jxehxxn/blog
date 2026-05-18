---
layout: post
title: "Ingress 보충 3: Ingress → Gateway API Migration"
date: 2026-05-19 03:00:00 +0900
categories: ingress gateway-api supplement
---

## Strategy
- Ingress 그대로 둠.
- 신규는 Gateway API.
- 점진 전환.

## 도구
- ingress2gateway (변환).

## 시점
v1 stable + controller 지원 확인.

## 결론
6~12개월 plan.
