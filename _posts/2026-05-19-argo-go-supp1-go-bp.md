---
layout: post
title: "ArgoCD Go 보충 1: Go Best Practice in ArgoCD"
date: 2026-05-19 04:00:00 +0900
categories: argocd go contributor supplement
---

## Error wrapping
`fmt.Errorf("foo: %w", err)`.

## Context propagation
모든 함수에 ctx.

## Goroutine 안전
sync.Mutex / channel.

## Logger
klog (K8s 표준).

## 결론
K8s ecosystem 표준 follow.
