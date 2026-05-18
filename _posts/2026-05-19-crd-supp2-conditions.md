---
layout: post
title: "CRD 보충 2: Status Condition Pattern"
date: 2026-05-19 06:00:00 +0900
categories: crd supplement
---

## 표준 Condition Type
- Ready: 사용 준비됨.
- Progressing: 진행 중.
- Degraded: 부분 문제.
- Available: K8s standard.

## 운영
- conditions 배열 (history 유지).
- LastTransitionTime 정확.
- Reason + Message 명확.

## 결론
K8s 표준. UI/tooling 호환.
