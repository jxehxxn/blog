---
layout: post
title: "ArgoCD 심화 Week 6: Resource Hooks 깊게"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced hooks platform senior-series
---

## 학습 목표

- PreSync/Sync/PostSync/SyncFail hook 깊게.
- Hook delete policy.
- Pattern (migration, smoke test, notification).

## 1. 4 위치 + Skip

- PreSync, Sync, PostSync, SyncFail.
- Skip: sync 단계 skip (사용 드물).

## 2. Hook Delete Policy

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

- `HookSucceeded`: 성공 후 삭제.
- `HookFailed`: 실패 후 삭제 (디버깅 위해 안 함도 OK).
- `BeforeHookCreation`: 다음 sync 때 이전 hook 삭제.

## 3. 일반 패턴

### DB Migration (PreSync)
```yaml
kind: Job
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
```

### Smoke Test (PostSync)
```yaml
kind: Job
metadata:
  annotations:
    argocd.argoproj.io/hook: PostSync
```

curl 또는 e2e test.

### Rollback Job (SyncFail)
```yaml
kind: Job
metadata:
  annotations:
    argocd.argoproj.io/hook: SyncFail
```

자동 rollback 또는 PagerDuty 호출.

## 4. Hook과 Wave 결합

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/sync-wave: "-1"
```

PreSync hook도 wave로 순서.

## 5. Skip Sync (sync 단계 skip)

```yaml
annotations:
  argocd.argoproj.io/sync-options: Skip
```

drift detection만, sync 안 함. data 자원 보호.

## 6. Hook 자체의 Resource 관리

Hook Job이 namespace에 누적되지 않도록 delete policy 필수.

## 7. 운영 함정

1. delete policy 빠짐 → Job 누적.
2. PreSync hook 실패 → 전체 sync 차단.
3. PostSync hook 결과 추적 어려움.
4. SyncFail hook 자체 실패.

## 8. 자가평가

### Q1. Hook 4 위치?
1. **PreSync/Sync/PostSync/SyncFail** 2. UI 3. 무관 4. 1개

**정답: 1.**

### Q2. BeforeHookCreation?
1. **다음 sync 때 이전 hook 삭제** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. PreSync 대표 용도?
1. **DB migration** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. PostSync 대표?
1. **smoke test, notification** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. SyncFail?
1. **자동 rollback / 알람** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## 9. 다음

[Week 7: SSO advanced].
