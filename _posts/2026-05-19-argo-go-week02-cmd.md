---
layout: post
title: "ArgoCD Go Week 2: cmd/ 구조"
date: 2026-05-19 04:00:00 +0900
categories: argocd go contributor senior-series
---

## Binary
- argocd (CLI).
- argocd-server.
- argocd-application-controller.
- argocd-repo-server.
- argocd-applicationset-controller.
- argocd-notifications.

각자 main.go.

## cobra
CLI/flag parsing.

## 자가평가
### Q1. binary 종류? **6개+**. 정답 1.
### Q2. CLI flag? **cobra**. 정답 1.
### Q3. main? **각 binary cmd/**. 정답 1.
### Q4. shared code? **pkg/, util/**. 정답 1.
### Q5. logging? **klog**. 정답 1.
