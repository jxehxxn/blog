---
layout: post
title: "Workflow 보충 4: Jenkins → Argo/Tekton 마이그레이션 플레이북"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series supplement
---

8주차에서 짧게 다룬 마이그레이션을 6개월 plan으로.

## 1. 사전 인벤토리

- Jenkins job 수, 종류 분류 (build, test, deploy, scheduled).
- plugin 사용 목록.
- 사용 secrets, credentials.
- 사용 노드 종류 (Windows, Mac, GPU).
- 평균 빌드 시간, peak concurrent.

이 인벤토리 없이 시작하면 실패.

## 2. 6개월 단계

### Month 1: Foundation
- Tekton/Argo Workflows 설치 + HA.
- 사내 catalog 기초 Task 5종 (git-clone, kaniko, golang-test, trivy, cosign).
- 신규 job 1~3개 새 도구로.
- 기존 Jenkins 그대로 운영.

### Month 2: Pilot
- 한 팀(5명 정도)의 job 5~10개 이주.
- 사용자 피드백 수집.
- catalog 보강.

### Month 3: Templates
- WorkflowTemplate / Pipeline 표준 3~5종 (Go service, JS service, ML, ...).
- 문서화 + 예시.
- 다른 팀 onboarding.

### Month 4: Scale
- 10~20 팀 이주.
- 평행 운영 (Jenkins + 새 도구).
- 모니터링 강화.

### Month 5: Sunset prep
- 남은 Jenkins job 모두 이주.
- 자체 도구(plugin) 호환 확인.
- 예외 처리.

### Month 6: Sunset
- Jenkins read-only → 완전 종료.
- 자산 정리 (서버, 백업).
- 사후 retrospective.

## 3. Mapping 표

| Jenkins | Argo/Tekton |
|---------|-------------|
| Job | WorkflowTemplate / Pipeline |
| Build | Task / Template |
| Pipeline (Jenkinsfile) | Pipeline (Tekton) / Workflow (Argo) |
| Parameter | param / parameters |
| Credential | Secret / ESO |
| Agent | Pod (자동) |
| Slave node | Node selector / tolerations |
| Trigger (cron, scm) | CronWorkflow / Argo Events / Tekton Triggers |
| Notification | exit handler / finally |
| Library (Groovy) | catalog Task |

## 4. 흔한 함정

1. **plugin 대체 누락**: Jenkins-only plugin을 미리 식별 + alternative 찾기.
2. **자격증명 분산**: ESO + Vault로 일원화.
3. **노드 의존성**: Windows/Mac 빌드는 K8s에서 어려움 → 별도 노드 또는 외부 SaaS 병행.
4. **사용자 저항**: 변화 관리 + 교육 필수.
5. **너무 빠른 sunset**: 6개월 권장, 4개월 미만 위험.

## 5. Quick Win 발굴

이주 초기에 보여줄 수 있는 가치:
- 빌드 시간 단축 (Kaniko cache).
- secret 안전성 (Vault).
- 자원 사용량 자동 scaling.

## 6. 사용자 교육

- 워크숍 2시간 × 4회.
- 사내 catalog 사용법.
- 디버깅 가이드 (logs, retry, suspend).
- ChatOps 봇.

## 7. Rollback 계획

위험: 이주 후 critical job 실패. plan:
- Jenkins 6개월 read-only 유지 (긴급 fallback).
- 매 단계 검증 (golden path).

## 8. 결론

Jenkins → 새 도구는 도구 변경이 아니라 **조직 변경**. 6개월 plan + 사용자 교육 + 단계별 검증. 빅테크 다수가 같은 길.
