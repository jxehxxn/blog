---
layout: post
title: "Workflow Week 8: Argo Workflows vs Tekton — 의사결정 기준"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series
---

## 학습 목표

- 두 도구의 본질적 차이.
- 6개 영역 정량 비교.
- 시나리오별 권장.
- 하이브리드 운영.

## 1. 한 줄

- **Argo Workflows**: 범용 워크플로 엔진. DAG + ML/data 강함.
- **Tekton**: CI/CD 특화. Task 표준화·재사용 강함.

## 2. 6 영역 비교

| 항목 | Argo Workflows | Tekton |
|------|----------------|--------|
| 주력 영역 | 범용 워크플로 (ML/data 포함) | CI/CD |
| 표현력 | DAG/steps/loop 풍부 | Pipeline DAG, finally |
| 재사용 | WorkflowTemplate | Task catalog (Hub) |
| 트리거 | Argo Events | Triggers |
| 공급망 보안 | 외부 통합 | Chains (내장) |
| 빅테크 사용 | Intuit, Spotify | Google, RedHat OpenShift |

## 3. 시나리오별 권장

### CI 빌드/테스트/배포
**Tekton**. 표준 catalog 풍부, Chains로 공급망 자동.

### ML training/data pipeline
**Argo Workflows**. DAG·loop·artifact 강력.

### 사내 self-service workflow 플랫폼
**Argo Workflows**. UI 강함, WorkflowTemplate 재사용.

### OpenShift 환경
**Tekton (Red Hat 기본)**.

### Google Cloud Build 호환
**Tekton (엔진)**.

## 4. 하이브리드 패턴

| 영역 | 도구 |
|------|------|
| 서비스 CI | Tekton |
| 데이터/ML | Argo Workflows |
| GitOps trigger | Argo Events |

같은 K8s 클러스터에서 둘 다 운영 가능. 자원·UI는 분리.

## 5. 학습 곡선

- Tekton: CI 경험자에게 자연스러움.
- Argo Workflows: 워크플로 엔진 처음이면 가파름 but 표현력 보상.

## 6. 운영 부담

비슷. 둘 다 K8s controller + UI + CRD. 정기 업그레이드, RBAC, secrets 등 같은 운영 영역.

## 7. 빅테크 선택 사례

- Intuit: Argo Workflows 모기업, 거의 모든 워크플로.
- Spotify: Argo Workflows + ArgoCD 통합.
- Red Hat: Tekton 표준 (OpenShift Pipelines).
- Google: Tekton (Cloud Build).
- Adobe: 둘 다 — 영역별 분리.

## 8. 한 회사에서 결정

질문 순서:
1. 주력 워크플로가 CI인가, 더 범용인가?
2. K8s 운영 역량은?
3. 사내 catalog 운영 의지는?
4. 외부 SaaS (GitHub Actions 등) 만족도는?

답에 따라 선택.

## 9. 마이그레이션 (Jenkins → Argo/Tekton)

자세한 절차는 [보충 4: Jenkins 마이그레이션].

요점:
- 한 번에 옮기지 말 것.
- 신규 job부터 새 도구.
- 6~12개월 병행.
- 사내 catalog/template 정착 후 본격 이주.

## 10. 자가평가 퀴즈

### Q1. CI 빌드/배포에 더 적합?
1. **Tekton**
2. Argo Workflows
3. 같음
4. 무관

**정답: 1.**

### Q2. ML/data 파이프라인에 더 적합?
1. **Argo Workflows**
2. Tekton
3. 같음
4. 무관

**정답: 1.**

### Q3. Tekton Chains의 가치?
1. **공급망 자동 서명·증명 (SLSA)**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q4. WorkflowTemplate 가치?
1. **재사용 가능한 사내 표준 절차**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. 빅테크 일반 패턴?
1. **영역별 분담 (CI=Tekton, 데이터=Argo)**
2. 하나만
3. UI
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 9: CI 패턴]에서 build/test/scan/sign/deploy 표준 패턴을 둘 다 다룹니다.
