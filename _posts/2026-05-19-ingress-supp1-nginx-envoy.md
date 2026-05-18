---
layout: post
title: "Ingress 보충 1: NGINX vs Envoy"
date: 2026-05-19 03:00:00 +0900
categories: ingress supplement
---

## NGINX
- 익숙, 안정.
- C 기반, 빠름.
- 동적 config 어려움 (reload).

## Envoy
- 동적 xDS.
- 풍부한 filter.
- 자원 약간 더.

## 결론
모던 mesh + dynamic은 Envoy. 단순은 NGINX.
