---
layout: post
title: "ArgoCD Week 12: 캡스톤 — Big Tech GitOps 플랫폼 풀스택 구축"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes capstone
---

## 캡스톤 개요

12주간 배운 모든 컴포넌트(Application/Set, sync 전략, 멀티클러스터, RBAC, 시크릿, Rollouts, observability, DR)를 묶어 **가상 빅테크 환경에 GitOps 플랫폼을 처음부터 구축**하는 프로젝트입니다.

## 1. 시나리오

당신은 가상 회사 **MegaCorp**에 들어온 Senior Platform Engineer입니다.

- Kubernetes 클러스터 12개 (4 리전 × 3 환경: dev/stage/prod).
- 마이크로서비스 250개, 200명 엔지니어, 12개 팀.
- HashiCorp Vault, Prometheus, Splunk 운영 중.
- SOC 2 Type II 인증 진행 중 (3개월 후 감사).
- 현재 배포는 Jenkins push 기반, 평균 배포 시간 30분, 사고율 5%.

CEO 요청: **"3개월 안에 GitOps 플랫폼 구축, 배포 시간 1/5로, 사고율 1% 이하로."**

## 2. 산출물

다음 5종을 모두 제출합니다.

1. **아키텍처 다이어그램**: Hub-Spoke 혼합, 12 cluster, 250 app fan-out 구조.
2. **ApplicationSet 모음 YAML**: List/Cluster/Matrix/PR 적어도 4종 사용.
3. **AppProject + RBAC CSV**: 12팀 격리, OIDC 그룹 매핑.
4. **Argo Rollouts + AnalysisTemplate**: canary + Prometheus 기반 자동 abort.
5. **DR 플랜 + 컴플라이언스 매핑**: SOC 2 CC8.1 매핑 표 + RTO/RPO 명세.

## 3. 평가 기준 (Grading Rubric)

채점 100점 만점.

### 기준 1: 아키텍처 적정성 (40점)
- [10] Hub-Spoke 혼합이 합리적 (Tier 분리)
- [10] ApplicationSet generator가 환경에 맞게 선택됨
- [10] Tenant 격리(AppProject + RBAC)가 leakage 없이 설계
- [10] HA + sharding + autoscaling이 정량적으로 명시

### 기준 2: 운영 완성도 (30점)
- [10] Sync wave/hook/retry/ignore-differences를 모두 운영 best practice로 적용
- [10] 시크릿 GitOps (ESO+Vault 또는 동등) 가 평문 0
- [10] Observability 4종 SLI/SLO 정의 + 알람 룰

### 기준 3: DR · 컴플라이언스 (30점)
- [10] DR 절차의 RTO/RPO 정량 + 복원 drill 결과
- [10] SOC 2 CC8.1 매핑 표가 evidence 위치까지 명시
- [10] Progressive delivery + 메트릭 기반 자동 abort로 사고율 < 1% 정량 근거

## 4. 채점 루브릭 + 3종 모범답안

별도 포스트 [ArgoCD Capstone: 채점 루브릭과 3종 모범답안] 참고.

## 5. 마무리

이 캡스톤을 완성하면 **빅테크에서 GitOps 플랫폼팀의 시니어 엔지니어로 첫 출근해도 즉시 운영을 시작할 수 있는 수준**입니다.

후속 학습 추천:
- ArgoCD 본가 contributor (regex/PR로 시작).
- Argo Workflows로 GitOps + CI 통합.
- Crossplane로 인프라까지 GitOps 화.
