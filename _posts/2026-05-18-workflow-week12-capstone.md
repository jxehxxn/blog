---
layout: post
title: "Workflow Week 12: 캡스톤 — 풀스택 CI/CD 플랫폼"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series capstone
---

## 캡스톤 개요

12주 학습을 묶어 **빅테크 CI/CD 플랫폼**을 처음부터 구축합니다.

## 1. 시나리오

MegaCorp:
- 200명, 12 팀, 250 마이크로서비스.
- 기존 Jenkins 500 job.
- K8s 클러스터 + ArgoCD 운영 중 (Series 1·2 가정).
- SOC 2 + 공급망 보안(SLSA Level 3) 요구.

CEO 요청: **"Jenkins 6개월 안에 sunset. 모든 CI K8s native, supply chain Level 3."**

## 2. 산출물

1. **아키텍처**: Argo Workflows (data/ML), Tekton (서비스 CI) 분담. Argo Events + ArgoCD 통합.
2. **표준 Pipeline**: build/test/scan/sign/deploy 5단계 Tekton Pipeline.
3. **사내 Catalog**: WorkflowTemplate / Tekton Task 표준 10+.
4. **공급망**: Cosign + Tekton Chains + Kyverno admission. SLSA Level 3 evidence.
5. **마이그레이션 플랜**: Jenkins → 새 시스템 6개월 단계별.

## 3. 평가 기준

총 100점.

### 기준 1: 아키텍처 (40)
- [10] Argo/Tekton 분담 명확
- [10] Argo Events + ArgoCD 통합
- [10] 표준 5단계 pipeline
- [10] HA 배포 + scale

### 기준 2: 보안 (30)
- [10] ESO + Vault secret
- [10] Sandbox + PSA restricted
- [10] SLSA Level 3 + Kyverno admission

### 기준 3: 거버넌스 (30)
- [10] 사내 Catalog 표준화
- [10] Audit 7년
- [10] 마이그레이션 6개월 plan 정량

## 4. 채점 + 모범답안

별도 포스트 [Workflow Capstone Rubric + 3종 모범답안] 참고.

## 5. 마무리

이 캡스톤 통과 시 빅테크 platform 팀에서 CI/CD 영역 시니어 즉시 시작 가능.
