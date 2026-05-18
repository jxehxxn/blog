---
layout: post
title: "ArgoCD로 알아보는 DevOps 실무: 12주 강의 계획서"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes
---

## 강의 개요

이 강의는 글로벌 빅테크(FAANG/MAGMA급) 환경에서 **ArgoCD**를 활용해 GitOps 기반 배포·운영 체계를 구축하는 12주 과정입니다. 단순히 `argocd app sync` 명령을 외우는 수준이 아니라, **수천 개 마이크로서비스·수십 개 Kubernetes 클러스터·여러 리전**에서 GitOps 플랫폼을 안전하게 굴리는 전체 라이프사이클을 다룹니다.

### 사전 요구 지식

- Kubernetes 기초 (Pod, Deployment, Service) (필수)
- YAML / Helm 기초 (필수)
- Git 워크플로우 이해 (필수)
- Linux 셸 사용 (필수)
- Go 또는 Python 기초 (커스텀 플러그인·webhook 주차)
- CI/CD 개념 (Jenkins/GitHub Actions 등 한 가지 이상 사용 경험)

### 학습 결과

12주를 마치면 수강생은 다음을 할 수 있습니다.

1. ArgoCD를 멀티테넌트·멀티클러스터 구조로 production-ready 배포
2. Application / ApplicationSet으로 1,000+ 앱을 선언적으로 관리
3. Sync waves, hooks, ignore differences를 활용한 안전한 변경 관리
4. SSO + RBAC + Project 기반 Tenant 격리 설계
5. Vault·SealedSecrets·SOPS 등으로 시크릿 GitOps화
6. Argo Rollouts와 결합한 Canary / Blue-Green / Progressive delivery
7. Prometheus·Grafana·OpenTelemetry로 GitOps 메트릭 운영
8. SOC 2 / 변경관리(Change Mgmt) / DR(재해복구) 매핑 + 감사 증빙 자동화

## 주차별 주제

| 주차 | 주제 | 형식 |
|------|------|------|
| 1주 | 오리엔테이션: 왜 GitOps인가, 왜 ArgoCD인가 | 강의 + 토론 |
| 2주 | ArgoCD 기초: 설치, 첫 Application, 첫 sync | 실습 |
| 3주 | Application / ApplicationSet — 선언적 앱 메타관리 | 실습 + 강의 |
| 4주 | Sync 전략 — wave, hook, retry, ignore differences | 실습 |
| 5주 | 리포지토리 도구 — Helm, Kustomize, Jsonnet, CMP | 실습 |
| 6주 | 멀티클러스터 / 멀티테넌트 운영 | 강의 + 실습 |
| 7주 | RBAC, SSO, Project — Tenant 격리 | 실습 |
| 8주 | 시크릿 GitOps — Vault, SealedSecrets, SOPS, Image Updater | 실습 |
| 9주 | Progressive Delivery — Argo Rollouts canary / blue-green | 실습 |
| 10주 | Observability — Prometheus, notifications, audit | 실습 |
| 11주 | Governance / DR / 변경관리·컴플라이언스 | 강의 |
| 12주 | 캡스톤: Big Tech GitOps 플랫폼 풀스택 구축 | 프로젝트 |

## 평가

- 매 주차 자가평가 퀴즈 (5문항, 정답+해설): 30%
- 주차별 실습 과제 제출: 30%
- 캡스톤 프로젝트: 40%

캡스톤은 3가지 채점 기준(아키텍처·운영·재해복구)으로 평가하며, 3종 모범답안(스타트업/스케일업/엔터프라이즈)이 제공됩니다.

## 보충 포스트 (Supplementary)

본 강의에서 "심화는 보충 자료 참조"라고 언급된 모든 주제는 별도 포스트로 작성됩니다.

- 보충 1: GitOps 이론 깊게 — pull vs push, reconciliation loop, CAP 관점
- 보충 2: Helm vs Kustomize — 정량 트레이드오프와 혼합 패턴
- 보충 3: ApplicationSet generator 8종 전수 분석
- 보충 4: Argo Rollouts AnalysisRun — Prometheus·Datadog·Web 분석 패턴

## 사용 도구 버전

- ArgoCD v2.11+ (2026년 기준 stable)
- Argo Rollouts v1.7+
- Kubernetes v1.29+
- Helm v3.14+, Kustomize v5.3+
- HashiCorp Vault 1.16
- Prometheus 2.50, Grafana 11

## 첫 주차 안내

다음 포스트 [Week 1: 오리엔테이션]에서는 **왜 빅테크가 Jenkins-style push 배포를 버리고 GitOps로 갈아탔는가**를 Intuit, Adobe, Spotify, Google 사례로 풀어봅니다.
