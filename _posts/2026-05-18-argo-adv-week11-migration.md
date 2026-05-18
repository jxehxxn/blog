---
layout: post
title: "ArgoCD 심화 Week 11: Migration — Flux / Spinnaker → ArgoCD"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced migration platform senior-series
---

## 학습 목표

- Flux → ArgoCD 절차.
- Spinnaker → ArgoCD.
- 단계별 plan.

## 1. Flux → ArgoCD

Flux CRD vs ArgoCD CRD:
- Kustomization → Application (path).
- HelmRelease → Application (chart).
- GitRepository → Application source.

## 2. 절차

1. ArgoCD parallel 설치.
2. 한 application 옮김 (Flux 비활성 + ArgoCD 생성).
3. drift 확인.
4. 모든 app 이주.
5. Flux 제거.

## 3. Spinnaker → ArgoCD

Spinnaker pipeline → ArgoCD + Argo Rollouts.
- canary stage → AnalysisTemplate.
- judgement → suspend.
- baking → CI 단계로.

## 4. 마이그레이션 도구

- `flux2argo`: 일부 자동 변환.
- 수동 작업 필요한 부분 많음.

## 5. 함정

1. 평행 운영 동안 자원 충돌.
2. label/annotation 다름.
3. Helm release naming.

## 6. Timeline

- 1주: 평가.
- 1개월: pilot.
- 3개월: 절반.
- 6개월: 완료.

## 7. 자가평가

### Q1. Flux Kustomization → ArgoCD?
1. **Application (path 기반)** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. Spinnaker canary → ArgoCD?
1. **AnalysisTemplate** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. 평행 운영 위험?
1. **자원 충돌** 2. 안전 3. UI 4. 무관

**정답: 1.**

### Q4. Timeline?
1. **6개월 정도** 2. 1주 3. 1년+ 4. 무관

**정답: 1.**

### Q5. 자동화 도구?
1. **일부만 (flux2argo) — 수동 검증 필수**
2. 모두 자동 3. UI 4. 무관

**정답: 1.**

## 8. 다음

[Week 12: 캡스톤].
