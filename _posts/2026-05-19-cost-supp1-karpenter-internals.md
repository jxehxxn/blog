---
layout: post
title: "Cost 보충 1: Karpenter Internals"
date: 2026-05-19 01:00:00 +0900
categories: cost karpenter supplement
---

## Reconciler
Pod scheduling 감시 → 최적 노드 결정.

## Provisioning
EC2 API 직접 호출 (ASG 없음).

## Consolidation
주기적 평가. 더 작은 노드로 통합.

## 결론
Cluster Autoscaler 대비 결정적 우위.
