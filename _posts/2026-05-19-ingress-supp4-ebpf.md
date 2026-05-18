---
layout: post
title: "Ingress 보충 4: eBPF Gateway (Cilium)"
date: 2026-05-19 03:00:00 +0900
categories: ingress cilium ebpf supplement
---

## Cilium Gateway
eBPF 기반 Gateway API 구현. 가장 빠름.

## 장점
- kube-proxy + Ingress 통합 (자원).
- L7 정책 내장.

## 단점
- Cilium CNI 의존.
- 신규.

## 결론
Cilium 사용 환경에선 자연.
