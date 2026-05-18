---
layout: post
title: "Workflow Week 1: 오리엔테이션 — 왜 K8s 위에 워크플로 엔진을?"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series
---

## 학습 목표

- Jenkins 시대의 한계를 안다.
- K8s native 워크플로의 4대 가치(declarative, isolation, scale, observability)를 안다.
- Argo Workflows와 Tekton의 차이 한 줄.
- Workflow engine과 CI 도구의 관계.

## 1. 비유 — "공방 vs 컨베이어 OS"

Jenkins: 작업장 한 가운데 거대한 작업대 + 모든 도구. 한 사람 한 사람 줄 서서 작업. 작업대가 망가지면 전체 정지.

K8s + Argo/Tekton: 컨베이어 OS. 각 작업이 자기 강의실(Pod)에서 격리되어 실행. OS가 자원을 분배. 한 작업의 실패가 다른 작업에 영향 0.

## 2. Jenkins 시대의 7가지 한계

1. **서버 폭증**: master/agent 노드를 직접 운영.
2. **격리 부족**: 같은 agent에 여러 job 동시 → 의존성·캐시 충돌.
3. **상태 보관 부담**: workspace, build cache 직접 관리.
4. **scale 어려움**: 트래픽 spike 시 수동 노드 추가.
5. **plugin 지옥**: 수백 plugin, 호환·보안 패치 부담.
6. **선언적 약함**: Groovy DSL이지만 명령형 잔재.
7. **관측 약함**: 어떤 step이 왜 느린지 추적 어려움.

## 3. K8s native 워크플로의 4대 가치

1. **Declarative**: YAML로 선언, GitOps 통합.
2. **Isolation**: 각 step이 Pod, 완전한 격리.
3. **Scale**: K8s가 노드 자동 확장.
4. **Observability**: K8s metric/log/event 그대로.

## 4. Argo Workflows — DAG 엔진 + CI 일부

- CNCF graduated. Argo Project (ArgoCD 자매).
- 강점: DAG/Step 풍부, ML/data 파이프라인 표준.
- CI 용도도 가능하지만, 순수 CI는 Tekton이 더 적합.

## 5. Tekton — Cloud Native CI

- CNCF, Google/IBM/Red Hat 주도.
- 강점: CI에 특화. Task/Pipeline 표준화.
- Argo Workflows보다 CI에 가깝게 설계.
- Tekton Chains로 공급망 보안 강력.

## 6. 한 줄 비교

- **Argo Workflows**: 워크플로 엔진. CI는 부분집합. ML/data 강함.
- **Tekton**: CI/CD 엔진. 표준화·재사용 강함.

빅테크 다수가 둘을 함께 또는 상황에 맞게 골라 씁니다.

## 7. CI 도구 시장에서의 위치

```
[자체 호스팅 OSS]
  - Jenkins (전통)
  - GitLab CI runner
  - Argo Workflows ← here
  - Tekton          ← here
  - Drone

[SaaS]
  - GitHub Actions
  - CircleCI
  - Travis CI
  - Buildkite
```

K8s에 이미 운영 중이면 Argo/Tekton이 자연스러움 + 비용 ↓.

## 8. 실제 사례

### Intuit (Argo 모기업)
사내 ML/data 파이프라인 거의 전체 Argo Workflows.

### CD Foundation
Tekton + Argo CD + Argo Workflows 통합 표준화 운동.

### Google Cloud
Cloud Build에 Tekton 엔진 채택.

## 9. 12주 로드맵

```
[기초]   1주 오리엔테이션 → 2주 Argo 기초 → 3주 artifacts
              ↓
[운영]   4주 패턴 → 5주 Events
              ↓
[비교]   6주 Tekton 기초 → 7주 Tekton 운영 → 8주 도구 비교
              ↓
[심화]   9주 CI 패턴 → 10주 GitOps 통합 → 11주 보안
              ↓
[캡스톤] 12주 풀스택
```

## 10. 실습 (필기형)

회사가 Jenkins로 500개 job 운영 중. K8s native로 이주 검토. 어떤 도구(Argo/Tekton/혼합)를 어떤 근거로 추천하시겠습니까? 200자.

## 11. 자가평가 퀴즈

### Q1. Jenkins 시대 가장 큰 한계?
1. **격리·scale·plugin 부담**
2. UI 색
3. 비용
4. 무관

**정답: 1.**

### Q2. K8s native 워크플로 4대 가치?
1. **Declarative, Isolation, Scale, Observability**
2. Mutable, Manual, Tight, Hidden
3. 무관
4. UI

**정답: 1.**

### Q3. Argo Workflows의 주력 영역?
1. **DAG 기반 ML/data 파이프라인 + CI 일부**
2. UI
3. 비용 절감
4. 무관

**정답: 1.**

### Q4. Tekton의 주력 영역?
1. **CI/CD에 특화**
2. ML
3. UI
4. 무관

**정답: 1.**

### Q5. 빅테크 선택 패턴?
1. **둘을 함께 또는 상황별 선택**
2. Jenkins만
3. Tekton만
4. 무관

**정답: 1.**

## 12. 다음 주차

[Week 2: Argo Workflows 기초]에서 Workflow CR + template 시스템을 다룹니다.
