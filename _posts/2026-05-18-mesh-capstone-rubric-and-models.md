---
layout: post
title: "Mesh Capstone: 채점 루브릭과 3종 모범답안"
date: 2026-05-18 19:00:00 +0900
categories: service-mesh platform senior-series capstone
---

## 1. 루브릭 (100)

### 기준 1: 아키텍처 (40)
### 기준 2: 보안 (30)
### 기준 3: 운영 (30)

(상세 Week 12 참조)

## 2. 모범답안 A — 스타트업

- Istio default profile, 1 cluster.
- mTLS PERMISSIVE.
- Telemetry default.
- Linkerd 검토 후 Istio 채택.

점수: ~55.

## 3. 모범답안 B — 스케일업

- Istio 3 cluster primary-remote.
- mTLS STRICT + AuthZ deny-by-default.
- Telemetry CR로 trace 1%.
- Argo Rollouts 통합 canary.
- canary upgrade plan.

점수: ~85.

## 4. 모범답안 C — 엔터프라이즈

- Istio multi-primary 5 region.
- Vault PKI + cert-manager.
- AuthZ + JWT + WAF integration.
- Ambient 일부 namespace 시범.
- 자체 contributor.

점수: ~95.

## 5. 결론

빅테크 senior mesh 운영의 표준 평가.
