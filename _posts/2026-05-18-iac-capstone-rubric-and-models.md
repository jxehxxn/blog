---
layout: post
title: "IaC Capstone: 채점 루브릭과 3종 모범답안"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series capstone
---

[Week 12 캡스톤] 평가 기준 + 3종 모범답안.

## 1. 루브릭 (100)

### 기준 1: 아키텍처 (40)
- [10] Terraform/Crossplane 분담 명확
- [10] State backend + lock + KMS
- [10] Module + Terragrunt DRY
- [10] Crossplane XRD/Composition 사내 추상화 3+

### 기준 2: 운영 (30)
- [10] Atlantis/Spacelift PR workflow
- [10] ArgoCD가 Crossplane CR sync
- [10] Drift detection + 자동 복구 정책

### 기준 3: Governance (30)
- [10] OPA policy 5+ + Infracost
- [10] Tagging 강제 + cost allocation 정확
- [10] SOC 2/PCI-DSS 매핑 + audit 7년

## 2. 모범답안 A — 스타트업

### 아키텍처
- Terraform만 (Crossplane은 K8s 보편화 전).
- S3 + DynamoDB backend, KMS encryption.
- Module 5종 (vpc, eks, rds, s3, iam).
- Terragrunt로 dev/prod 디렉토리 분리.

### 운영
- GitHub Actions로 terraform plan/apply.
- Atlantis는 도입 비용 부담 (소규모).
- Drift detection cron 일 1회.

### Governance
- OPA 3개 (S3 public 차단, tag 강제, IAM `*` 금지).
- Infracost CI 통합.
- SOC 2 미준비.

점수: 약 62/100. 통과지만 Crossplane 영역 누락.

## 3. 모범답안 B — 스케일업

### 아키텍처
- 하이브리드: Terraform(기반) + Crossplane(앱별).
- State backend HA (S3 cross-region replicate + KMS).
- Module 15+ + Terragrunt + 사내 module registry.
- Crossplane XRD 3+ (PostgreSQL, Bucket, IAMRole).

### 운영
- Atlantis PR workflow.
- ArgoCD가 Crossplane Configuration + Claim sync.
- Drift detection: Terraform cron + Crossplane continuous.

### Governance
- OPA 7개 + Infracost.
- default_tags 모든 provider.
- SOC 2 CC8.1 매핑 + audit 7년 S3 WORM.

점수: 약 86/100. 시니어 수준.

## 4. 모범답안 C — 엔터프라이즈

### 아키텍처
- 멀티클라우드 (AWS+GCP). 50+ 클러스터.
- State 분할 (per-region, per-service).
- Module monorepo + 자동 release.
- Crossplane XRD 10+ + Function 사용. Configuration package OCI 배포.

### 운영
- Spacelift (SaaS) + Atlantis 병행.
- ArgoCD ApplicationSet으로 Crossplane CR 멀티클러스터 sync.
- Drift: 자동 복구 + SIEM 적재.

### Governance
- OPA 20+ (SOC 2/PCI/HIPAA 매핑).
- Infracost 모든 PR + cost owner 별도 승인.
- Audit 7년 WORM + 분기 감사 보고 자동.

점수: 약 96/100. 빅테크 운영팀 수준.

## 5. 평가 시뮬레이션 (실패)

- Terraform local state (S3 X)
- Crossplane 없음
- OPA 미적용

점수: ~25/100. 재제출.

## 6. 마무리

루브릭은 빅테크 platform 시니어 평가의 표준. 약점 진단이 성장의 출발.
