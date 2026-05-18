---
layout: post
title: "Ingress Week 1: Ingress 한계와 Gateway API 등장"
date: 2026-05-19 03:00:00 +0900
categories: ingress senior-series
---

## Ingress의 한계
- vendor 별 annotation 난립.
- L7 표현력 부족.
- TCP/UDP/gRPC 약함.
- 권한 분리 부재.

## Gateway API
GatewayClass + Gateway + Route 분리. 인프라/플랫폼/앱팀 책임 분리.

## 자가평가
### Q1. Ingress 한계? **annotation 난립**. 정답 1.
### Q2. Gateway API 가치? **책임 분리 + 표현력**. 정답 1.
### Q3. 3계층? **GatewayClass/Gateway/Route**. 정답 1.
### Q4. Annotation 표준화 어려운 이유? **vendor마다 다름**. 정답 1.
### Q5. 미래? **Gateway API 표준**. 정답 1.
