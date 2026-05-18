---
layout: post
title: "Policy Week 12: 캡스톤 — 풀스택 Policy 플랫폼"
date: 2026-05-18 20:00:00 +0900
categories: policy opa kyverno platform senior-series capstone
---

## 시나리오

MegaCorp:
- 5 cluster, 12 팀, 250 서비스.
- SOC 2 + 공급망 (SLSA Level 3).
- 정책 30+.

## 산출물

1. 아키텍처: Kyverno (mutation/image) + OPA (Terraform CI). 둘 다 GitOps.
2. 정책 30+: tagging, image, security context, NetworkPolicy 자동 생성.
3. 라이프사이클 plan: dryrun → enforce.
4. Audit + Compliance report.
5. multi-cluster ApplicationSet.

## 루브릭

총 100. (40 아키텍처, 30 운영, 30 governance)

## 모범답안 3종

별도 포스트. 스타트업/스케일업/엔터프라이즈.

## 마무리

빅테크 정책 운영 시니어 즉시 가능.
