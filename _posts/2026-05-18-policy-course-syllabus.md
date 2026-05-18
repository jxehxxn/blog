---
layout: post
title: "Policy as Code 깊이 알기 (Senior Series 6/19): OPA Gatekeeper + Kyverno"
date: 2026-05-18 20:00:00 +0900
categories: policy opa gatekeeper kyverno platform senior-series
---

## 강의 개요

빅테크 Senior Platform 수준의 **Policy as Code** 12주 과정. OPA Gatekeeper + Kyverno 모두 깊게 + 의사결정 기준.

전제 비유: **Policy as Code = "법전이자 자동 경찰관"**. 정책을 코드(YAML/Rego)로 두고, K8s admission controller가 자동으로 적용. 위반은 차단하거나 audit log.

## 사전 지식

- Kubernetes 기초 (필수)
- YAML / 정규표현식
- 가벼운 프로그래밍 (Rego 학습용)

## 학습 결과

12주 후:
1. Gatekeeper + Kyverno production 배포.
2. Rego 자유 작성.
3. ConstraintTemplate, Kyverno policy 작성.
4. Mutation + Image admission.
5. Audit + GitOps lifecycle.

## 주차

| 주 | 주제 |
|----|------|
| 1 | 오리엔테이션 — Policy as Code |
| 2 | OPA 기초 + Rego 첫걸음 |
| 3 | Gatekeeper 아키텍처 + ConstraintTemplate |
| 4 | Kyverno 기초 |
| 5 | OPA vs Kyverno 비교 |
| 6 | Rego 깊게 |
| 7 | Mutation (Kyverno) |
| 8 | Image admission + 공급망 |
| 9 | Audit + Violations 관리 |
| 10 | External data + Sync |
| 11 | GitOps + 라이프사이클 |
| 12 | 캡스톤 |

## 보충

- 보충 1: Rego 심화
- 보충 2: Kyverno mutation 패턴
- 보충 3: Policy catalog 운영
- 보충 4: Performance impact

## 다음

[Week 1]에서 Policy as Code의 본질 정의.
