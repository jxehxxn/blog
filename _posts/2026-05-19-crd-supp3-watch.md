---
layout: post
title: "CRD 보충 3: Watch + Secondary Cache"
date: 2026-05-19 06:00:00 +0900
categories: crd supplement
---

## Owns vs Watches
- Owns: 자식 자원 watch.
- Watches: 임의 자원 watch + custom mapping.

## Secondary cache
controller-runtime이 자동 informer 운영.

## 결론
큰 cluster에서 watch design이 성능 핵심.
