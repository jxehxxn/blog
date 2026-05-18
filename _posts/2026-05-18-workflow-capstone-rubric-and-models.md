---
layout: post
title: "Workflow Capstone: 채점 루브릭과 3종 모범답안"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series capstone
---

[Week 12 캡스톤] 평가 기준 + 3종 모범답안.

## 1. 루브릭 (100)

### 기준 1: 아키텍처 (40)
- [10] Argo/Tekton 분담
- [10] Argo Events + ArgoCD 통합
- [10] 표준 5단계 pipeline
- [10] HA + scale

### 기준 2: 보안 (30)
- [10] ESO + Vault secret
- [10] Sandbox + PSA restricted
- [10] SLSA Level 3 + Kyverno admission

### 기준 3: 거버넌스 (30)
- [10] 사내 Catalog
- [10] Audit 7년
- [10] 마이그레이션 6개월 plan 정량

## 2. 모범답안 A — 스타트업

- Tekton만 (Argo Workflows 도입 X, CI만 필요).
- 5단계 standard pipeline.
- SA 분리 + secretRef.
- Trivy + Cosign 기본.
- Audit 1년.
- Jenkins 부재 → migration 무관.

점수: ~65. 통과지만 ML/data 영역 누락.

## 3. 모범답안 B — 스케일업

- Tekton(서비스 CI) + Argo Workflows(data/ML).
- Catalog 10+ Task.
- ESO + Vault.
- PSA restricted + gVisor 일부.
- SLSA Level 3 + Chains.
- Audit 7년.
- Jenkins 6개월 sunset plan.

점수: ~88.

## 4. 모범답안 C — 엔터프라이즈

- Tekton + Argo Workflows + Argo Events 모두.
- Catalog 50+ + 사내 Backstage 통합.
- HA + sharding (controller 5+, namespace 분담).
- Vault dynamic + IRSA.
- SLSA Level 4 시도 (reproducible build).
- Admission policy 20+.
- Audit SIEM 7년 + WORM.
- Jenkins sunset 완료 + 자체 metric/dashboard.

점수: ~96.

## 5. 평가 시뮬레이션 (실패)

- Jenkins 그대로 + Argo만 일부.
- Catalog 0.
- Cosign 미사용.

점수: ~30. 재제출.

## 6. 결론

빅테크 platform 시니어 평가의 표준. 약점 진단이 본질.
