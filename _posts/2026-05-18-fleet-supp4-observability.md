---
layout: post
title: "Fleet 보충 4: Fleet Observability"
date: 2026-05-18 23:30:00 +0900
categories: fleet observability supplement
---

## Metric
- ManagedCluster status.
- CAPI Machine state.
- ArgoCD app sync.

## Dashboard
Grafana fleet view: cluster별 status + lag.

## Alert
cluster down, drift, upgrade failed.

## Multi-cluster aggregation
Thanos/Mimir로 모든 cluster metric 통합.

## 결론
50+ cluster 운영은 관측이 핵심.
