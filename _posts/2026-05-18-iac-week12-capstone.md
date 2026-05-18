---
layout: post
title: "IaC Week 12: 캡스톤 — 멀티클라우드 GitOps 인프라 풀스택"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series capstone
---

## 캡스톤 개요

12주의 모든 학습(Terraform 모듈, state, Atlantis, Crossplane Composition, drift, governance)을 묶어 **가상 빅테크의 멀티클라우드 인프라**를 IaC로 구축합니다.

## 1. 시나리오

**MegaCorp** 가상 회사:
- AWS + GCP 멀티클라우드.
- 4 region, dev/stage/prod.
- 200명 엔지니어, 12 팀.
- K8s 클러스터 12개.
- SOC 2 Type II + PCI-DSS 인증 진행 중.

CEO 요청: **"콘솔 운영 0% 목표. 모든 인프라 PR 기반. 신규 팀 onboarding 1일 이내."**

## 2. 산출물

1. **아키텍처 다이어그램**: Terraform(기반) + Crossplane(앱별) 하이브리드. Atlantis/Spacelift + ArgoCD.
2. **Terraform 모듈 + Terragrunt**: VPC, EKS, IAM, RDS, S3 등 핵심 자원.
3. **Crossplane XRD + Composition**: PostgreSQL, Bucket, ServiceAccount 등 사내 추상화 3종 이상.
4. **GitOps workflow**: Atlantis(Terraform PR) + ArgoCD(Crossplane CR sync).
5. **Governance**: OPA policy 5개 이상, Infracost CI 통합, SOC 2 CC8.1 매핑 표.

## 3. 평가 기준 (Rubric)

총 100점.

### 기준 1: 아키텍처 (40)
- [10] Terraform/Crossplane 분담이 명확
- [10] State backend + lock + KMS
- [10] Module + Terragrunt DRY
- [10] Crossplane XRD + Composition 사내 추상화

### 기준 2: 운영 (30)
- [10] Atlantis or Spacelift PR workflow
- [10] ArgoCD가 Crossplane CR sync
- [10] Drift detection + 자동 복구 정책

### 기준 3: Governance (30)
- [10] OPA policy 5+ + Infracost
- [10] Tagging 강제 + cost allocation 정확
- [10] SOC 2 / PCI-DSS 매핑 + audit evidence 7년

## 4. 채점 루브릭 + 3종 모범답안

별도 포스트 [IaC Capstone Rubric + 3종 모범답안] 참고.

## 5. 마무리

이 캡스톤을 통과하면 빅테크에서 **인프라를 100% IaC로 운영하는 platform 시니어**로 즉시 일할 수 있는 수준.
