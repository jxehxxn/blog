---
layout: post
title: "Fleet 보충 1: CAPI Internals"
date: 2026-05-18 23:30:00 +0900
categories: fleet capi supplement
---

## Controller 구성
- cluster-api: 일반 controller.
- bootstrap-kubeadm, control-plane-kubeadm.
- infrastructure-aws/gcp/azure.

## Reconciliation
각 CR 변경 시 reconcile. Cloud API 호출.

## Webhook
admission validation.

## 결론
표준 K8s operator 패턴. Course 17과 같은 구조.
