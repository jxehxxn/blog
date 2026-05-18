---
layout: post
title: "ArgoCD 심화 Week 2: Source Hydrator (v2.13+)"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced platform senior-series
---

## 학습 목표

- Source Hydrator의 개념.
- Image Updater와의 차이.
- 운영 패턴.

## 1. 비유 — "녹음실 vs 마스터링실"

원본 트랙(source) → 마스터링(hydrate) → 배포(deploy). 마스터링 결과도 git에 저장 → 누구나 검토.

## 2. 기존 모델 한계

ArgoCD가 직접 helm template / kustomize build → 결과는 ArgoCD 내부 cache만. git history에 없음.

문제:
- Diff 시각화 불편.
- "지금 실제로 적용되는 manifest" 불투명.
- audit 어려움.

## 3. Source Hydrator

`source` (high-level) → ArgoCD가 hydrate → `targetRevision` 별도 brunch에 결과 commit.

```yaml
spec:
  sourceHydrator:
    drySource:
      repoURL: https://github.com/mycorp/manifests.git
      targetRevision: main
      path: apps/myapp
    syncSource:
      targetBranch: env/prod
      path: apps/myapp
    hydrateTo:
      targetBranch: env/prod-hydrated
```

흐름:
1. main brunch에 source 변경.
2. ArgoCD가 hydrate (helm/kustomize).
3. `env/prod-hydrated` brunch에 commit.
4. ArgoCD가 그 brunch sync.

## 4. 장점

- 실제 적용 manifest가 git에.
- diff 쉬움 (brunch 간).
- audit + rollback 명확.

## 5. Image Updater와 차이

- Image Updater: image tag만 자동 업데이트.
- Source Hydrator: manifest 전체 변환 결과 commit.

둘 다 가능. 통합 사용.

## 6. 운영

- hydrated brunch는 ArgoCD 전용 (사람 직접 commit X).
- branch protection.
- ArgoCD 인증 token에 push 권한.

## 7. 자가평가

### Q1. Source Hydrator 가치?
1. **hydrated manifest를 별도 brunch에 → 투명/audit** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. Image Updater와 차이?
1. **Image Updater는 tag만, Hydrator는 manifest 전체** 2. 같음 3. UI 4. 무관

**정답: 1.**

### Q3. hydrated brunch에 사람 직접 commit?
1. **금지 — ArgoCD 전용** 2. 허용 3. UI 4. 무관

**정답: 1.**

### Q4. 권한 요구?
1. **ArgoCD가 brunch push 권한** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. 도입 시점?
1. **v2.13+ + 검토 완료 후** 2. 즉시 3. UI 4. 무관

**정답: 1.**

## 8. 다음

[Week 3: CMP].
