---
layout: post
title: "Policy Capstone: 채점 루브릭과 3종 모범답안"
date: 2026-05-18 20:00:00 +0900
categories: policy platform senior-series capstone
---

## 루브릭 (100)

- 기준 1: 아키텍처 (40)
- 기준 2: 운영 (30)
- 기준 3: governance (30)

## 모범답안 A — 스타트업

- Kyverno만, 정책 5개.
- enforce 직접.
- audit 미사용.

점수: ~50.

## 모범답안 B — 스케일업

- Kyverno + OPA conftest.
- 정책 20+ + lifecycle.
- ApplicationSet multi-cluster.
- Policy Reporter dashboard.

점수: ~85.

## 모범답안 C — 엔터프라이즈

- Kyverno + OPA Gatekeeper 분담.
- 정책 50+ + library 통합.
- Catalog + Backstage.
- SLSA Level 3 + compliance report 자동.

점수: ~95.

## 결론

빅테크 정책 운영 senior 표준.
