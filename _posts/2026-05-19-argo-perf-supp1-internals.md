---
layout: post
title: "ArgoCD Perf 보충 1: Controller Internals"
date: 2026-05-19 02:00:00 +0900
categories: argocd performance supplement
---

## Reconcile loop
1. Application list (watch).
2. Diff (desired vs live).
3. Sync (필요 시).
4. Status update.

## Bottleneck
Diff 계산 + Sync. Pod이 많은 app일수록.

## Optimization
ApplyOutOfSyncOnly, ServerSideApply.
