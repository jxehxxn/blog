---
layout: post
title: "ArgoCD 심화 Week 1: 심화 영역 overview"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced platform senior-series
---

## 학습 목표

- ArgoCD 기본 → 심화 차이.
- 빅테크가 확장하는 5 영역.
- 12주 로드맵.

## 1. 비본 → 심화

기본: Application/Set, sync 정책, RBAC.
심화: 확장 (plugin, custom controller), API 직접 통합, 운영 자동화.

## 2. 5 영역

1. **Manifest generation 확장** (CMP).
2. **App 생성 자동화 확장** (custom ApplicationSet generator).
3. **Source 처리 새 모델** (Source Hydrator).
4. **Notification customization**.
5. **API 직접 통합** (사내 IDP, Backstage).

## 3. 빅테크 시나리오

- Spotify: Backstage 통합 → API 호출.
- Adobe: 사내 manifest 도구 → CMP.
- Intuit: 자체 ApplicationSet generator.

## 4. 로드맵

```
[확장]  2 Hydrator → 3 CMP → 4 Generator → 5 Notif → 6 Hooks
[심화]  7 SSO → 8 Operator
[활용]  9 패턴 → 10 API → 11 Migration → 12 캡스톤
```

## 5. 자가평가

### Q1. 심화 영역 5?
1. **CMP/Generator/Hydrator/Notif/API** 2. UI 3. 무관 4. CPU

**정답: 1.**

### Q2. CMP 가치?
1. **임의 manifest 도구 통합 (ytt, cdk8s 등)** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. ApplicationSet plugin generator?
1. **자체 generator로 사내 데이터 → Application** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. Source Hydrator (v2.13+)?
1. **manifest 변환을 별도 git branch에 hydrate** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. 빅테크 API 통합?
1. **Backstage/IDP에서 ArgoCD API 호출** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## 6. 다음

[Week 2: Source Hydrator].
