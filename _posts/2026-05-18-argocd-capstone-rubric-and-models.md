---
layout: post
title: "ArgoCD Capstone: 채점 루브릭과 3종 모범답안 (스타트업/스케일업/엔터프라이즈)"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes capstone
---

본 포스트는 [Week 12 캡스톤]의 평가 기준과 모범답안 3종을 상세히 풉니다.

## 1. 채점 루브릭 — 총 100점

### 기준 1: 아키텍처 적정성 (40점)

| 항목 | 점수 | 평가 |
|------|------|------|
| Hub-Spoke 혼합 | 10 | Tier 분리 명확 |
| ApplicationSet generator 선택 | 10 | 환경에 맞는 generator 4종 이상 |
| Tenant 격리 | 10 | leakage 없는 AppProject + RBAC |
| HA + sharding + autoscaling | 10 | 정량 명세 (replicas, HPA) |

### 기준 2: 운영 완성도 (30점)

| 항목 | 점수 | 평가 |
|------|------|------|
| Sync 전략 best practice | 10 | wave/hook/retry/ignore 모두 |
| 시크릿 GitOps | 10 | 평문 0, ESO+Vault 등 |
| Observability | 10 | SLI/SLO 4종 + 알람 |

### 기준 3: DR · 컴플라이언스 (30점)

| 항목 | 점수 | 평가 |
|------|------|------|
| DR 절차 | 10 | RTO/RPO 정량 + drill 결과 |
| SOC 2 CC8.1 매핑 | 10 | evidence 위치까지 |
| Progressive delivery 자동 abort | 10 | 사고율 < 1% 정량 근거 |

## 2. 모범답안 A — 스타트업 (50명, 3 클러스터)

### 아키텍처
- Hub-Spoke 단일 ArgoCD가 dev/stage/prod 3 클러스터 관리.
- ApplicationSet: List generator로 환경 분기, PullRequest로 preview env.
- Tenant 격리: 팀이 2개라 AppProject 2개로 충분.
- HA: server 2, controller 1, repo 2. 작은 규모.

### 운영
- Sync: wave 3단계, PreSync DB migration, PostSync Slack.
- 시크릿: SealedSecrets (Vault 도입 비용 부담).
- Observability: 공식 Grafana dashboard + Slack 알람.

### DR / 컴플라이언스
- DR: Velero로 daily backup, RTO 1h, drill 분기 1회.
- SOC 2 미준비, 도입 후 평가 예정.
- Argo Rollouts: 간단 canary 2단계, smoke test만 분석.

### 강약점
- 강점: ROI 명확, 빠른 도입.
- 약점: Vault 미사용, 정량 분석 부족.

## 3. 모범답안 B — 스케일업 (200명, 12 클러스터)

### 아키텍처
- 하이브리드: prod 클러스터마다 자체 ArgoCD, dev/stage는 hub-spoke.
- ApplicationSet: List + Cluster + Matrix + PullRequest 4종.
- Tenant 격리: AppProject 12개 (팀별), OIDC 그룹 매핑.
- HA: server 3, controller 3 (sharded), repo 5 autoscale, redis HA.

### 운영
- Sync: wave 5단계 표준화, PreSync migration, PostSync smoke + Slack/PagerDuty.
- 시크릿: ESO + Vault dynamic secret. K8s Secret은 ignoreDifferences.
- Observability: 4종 SLI/SLO + Notifications + OTel tracing.

### DR / 컴플라이언스
- DR: Active-Passive (prod 2 리전), RTO 30분, drill 월 1회.
- SOC 2 CC8.1 매핑 완비, evidence S3 WORM 7년.
- Argo Rollouts: canary 4단계 + Prometheus 3 메트릭(error/latency/business), 자동 abort 1년 추적.

### 강약점
- 강점: 자동화 완성도, 메트릭.
- 약점: Active-Active 미도입 (prod down 시 RTO 30분).

## 4. 모범답안 C — 엔터프라이즈 (5,000명, 50+ 클러스터)

### 아키텍처
- Decentralized 우선: 각 prod 클러스터에 자체 ArgoCD.
- Tier 1 dev/stage는 hub-spoke 클러스터에 통합.
- ApplicationSet: 8종 모두 사용. SCM Provider로 1,000+ repo 자동 인지.
- Tenant 격리: AppProject 50+ (팀별 + 환경별), OIDC + JIT access.
- HA: 모든 component HA, controller sharding 10 replicas, repo HPA 20+.

### 운영
- Sync: 정밀 wave + hook + AnalysisRun + custom CMP 다수.
- 시크릿: ESO + Vault dynamic 전체. Image Updater PR-only.
- Observability: 4종 SLI/SLO + OTel + 자체 ArgoCD SLO 보드(month-over-month error budget).

### DR / 컴플라이언스
- DR: Active-Active 3 리전, RTO 0초, drill 주 1회 game day.
- SOC 2 + ISO 27001 + PCI 모두 매핑, audit log Splunk + S3 WORM 7년.
- Argo Rollouts: 4-layer analysis (smoke/error/latency/business), 자동 abort로 사고율 < 0.5%.

### 강약점
- 강점: 거의 모든 영역 자동화.
- 약점: 운영 비용 큼, 도구의 도구 (ArgoCD 자체 보안 통제)가 별도 과제.

## 5. 평가 시뮬레이션

가상의 제출본:
- ArgoCD HA 없음 (단일 replica)
- ApplicationSet List만 사용
- AppProject 없이 default
- 시크릿 SealedSecrets만
- DR drill 미시행

점수:
- 기준 1: Hub-Spoke(6) + Generator(3) + 격리(0) + HA(0) = **9/40**
- 기준 2: Sync(5) + 시크릿(5) + Obs(3) = **13/30**
- 기준 3: DR(2) + 매핑(0) + Rollouts(3) = **5/30**
- 합계: **27/100** → 재제출.

피드백: "HA 부재, 격리 미흡, DR drill 부재가 치명적. 캡스톤 통과 어려움."

## 6. 마무리

이 루브릭과 모범답안은 빅테크 플랫폼팀 시니어 면접/평가 기준에 가깝습니다. 점수보다 **약점 진단 능력**이 핵심.
