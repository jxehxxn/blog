---
layout: post
title: "Terraform + Crossplane 깊이 알기 (Senior Series 2/19): 12주 강의 계획서"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series
---

## 이 강의가 누구를 위한 것인가

학부생~주니어가 **빅테크 Senior Platform Engineer 수준의 IaC(Infrastructure as Code) + GitOps for Cloud** 이해에 도달하기 위한 12주 과정. Terraform과 Crossplane을 모두 깊게 다루고, **언제 어떤 도구를 쓰는가**의 판단력을 길러줍니다.

전제 비유: **클라우드 인프라는 도시이고, IaC는 "도시 설계도".** 콘솔 클릭으로 도시를 짓는 건 수기로 집을 하나씩 그리는 격. IaC는 도면을 두고 "이렇게 지어줘"라고 선언. 도면이 곧 진실이라서 다시 짓거나(rebuild), 다른 곳에 짓거나(replicate), 누가 만졌는지 확인(audit) 모두 가능합니다.

## 사전 지식

- Linux 셸 (필수)
- AWS/GCP/Azure 중 하나의 콘솔 경험 (필수)
- Kubernetes 기초 — 6주차 이후 (필수, Course 1 추천)
- Git 기초 (필수)
- HCL/YAML 읽기 가능

## 학습 결과

12주 후 다음이 가능해야 합니다.

1. Terraform으로 AWS/GCP 멀티 리전 인프라를 모듈화해 배포.
2. State 파일의 backend·locking·workspace 운영.
3. Atlantis/Spacelift 같은 GitOps CI로 Terraform PR 워크플로 운영.
4. Crossplane으로 같은 인프라를 K8s CRD로 관리.
5. Composition으로 사내 표준 인프라 패키지(XRD) 작성.
6. Terraform과 Crossplane 사이의 정확한 의사결정 기준 적용.
7. Drift 자동 감지/복구 파이프라인 구축.

## 주차 계획

| 주 | 주제 | 형식 |
|----|------|------|
| 1 | 오리엔테이션 — 왜 IaC인가, 클릭 시대의 종말 | 강의 |
| 2 | Terraform 기초 — provider, resource, plan/apply | 실습 |
| 3 | State 깊게 — backend, locking, workspaces, import | 실습 |
| 4 | Modules + Terragrunt — DRY at scale | 실습 |
| 5 | Terraform workflow — Atlantis/Spacelift/Env0, GitOps | 실습 |
| 6 | Crossplane 소개 — 클라우드를 K8s CRD로 | 실습 |
| 7 | Composition + XRD — 사내 인프라 패키지화 | 실습 |
| 8 | Crossplane vs Terraform — 의사결정 기준 | 강의 |
| 9 | Crossplane 운영 — providers, packages, RBAC | 실습 |
| 10 | Drift detection & remediation | 실습 |
| 11 | Governance · cost · compliance | 강의 |
| 12 | 캡스톤 — 멀티클라우드 GitOps 인프라 풀스택 | 프로젝트 |

## 보충 포스트

- 보충 1: State 파일 깊게 — lock race, 복구, secret 처리
- 보충 2: Module 디자인 패턴 — composition, anti-patterns, registry
- 보충 3: Crossplane Composition Functions — v1.14+ 혁명
- 보충 4: Terraform-Crossplane 하이브리드 — 두 도구 공존 패턴

## 평가

- 자가평가 30%
- 실습 30%
- 캡스톤 40% (3종 모범답안: 단일클라우드/멀티클라우드/엔터프라이즈)

## 다음 주차

[Week 1: 오리엔테이션]에서 클릭 운영의 한계와 IaC의 본질, Terraform/Crossplane의 포지셔닝을 정리합니다.
