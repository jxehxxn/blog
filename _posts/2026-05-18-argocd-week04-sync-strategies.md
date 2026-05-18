---
layout: post
title: "ArgoCD Week 4: Sync 전략 — Wave, Hook, Retry, Ignore Differences로 안전한 변경 관리"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes
---

## 학습 목표

- Sync wave로 리소스 적용 순서를 제어한다.
- Sync hook(PreSync/Sync/PostSync/SyncFail)으로 마이그레이션·검증·알림을 자동화한다.
- Retry/backoff로 일시적 장애를 흡수한다.
- IgnoreDifferences로 controller 변경(HPA replicas 등)을 무시한다.

## 1. Sync Wave — 순서가 있는 변경

비유: 새 식당 오픈을 위해 (1) 전기, (2) 가스, (3) 가구, (4) 음식 순서대로 들여놓아야 합니다. 동시에 모두 들여놓으면 식기가 가스 공사 중에 다 깨집니다.

ArgoCD에서:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"  # 음수 = 먼저
```

- Wave 값 작은 것부터 큰 것 순서대로 적용.
- 같은 wave는 병렬.
- 기본 0.

빅테크 패턴: ConfigMap/Secret(-2) → CRD(-1) → Deployment(0) → Job(1) → Service/Ingress(2).

## 2. Sync Hook — 4가지 위치

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

- `PreSync`: sync 시작 전. 예: DB migration Job.
- `Sync`: sync 도중. 예: validation Job.
- `PostSync`: sync 완료 후. 예: smoke test, 알림 webhook.
- `SyncFail`: sync 실패 시. 예: rollback Job, on-call 페이지.

빅테크 표준: PreSync에 DB migration, PostSync에 smoke test + Slack notification.

### 예: DB migration with PreSync

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-db
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: mycorp/migrate:1.2.3
          command: ["./migrate.sh"]
      restartPolicy: Never
```

`BeforeHookCreation`: 다음 sync 때 이전 Job을 지우고 새로 만듦. 결과는 ArgoCD UI에서 추적.

## 3. Retry & Backoff

일시 장애(image pull 실패, webhook timeout)는 자동 재시도.

```yaml
syncPolicy:
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

- 5초 → 10초 → 20초 → 40초 → 80초 → max 180초.
- 5회 모두 실패하면 Application은 Degraded.

빅테크 권장: limit 5, factor 2, max 3m. 너무 짧으면 일시 장애도 fail, 너무 길면 진짜 사고도 fail이 늦음.

## 4. IgnoreDifferences — Controller 변경 무시

HPA가 replicas를 자동 조정하는 경우, Git의 `replicas: 3`과 live `replicas: 7`이 늘 다릅니다. 이를 매번 sync로 되돌리면 HPA 의미가 없어집니다.

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

빅테크 운영 패턴:
- HPA가 다루는 replicas
- ServiceAccount의 secrets 자동 채움
- VPA의 resources 권고
- 모두 ignoreDifferences로 제외.

## 5. SyncOptions — 미세 조정

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    - Validate=false        # kubectl validate 끔(CRD 의존성 회피)
    - ApplyOutOfSyncOnly=true  # 차이나는 것만 apply
    - ServerSideApply=true     # SSA 활성
    - Replace=true             # apply 대신 replace
```

빅테크에서 가장 많이 쓰는 옵션:
- `ServerSideApply=true`: 큰 리소스(특히 CRD)에서 client-side apply의 metadata.annotations.last-applied 한계 우회.
- `PruneLast=true`: 삭제는 가장 마지막에. 순환 의존 정리.

## 6. 실제 사례 — Adobe의 sync wave 정책

Adobe 발표 자료에서 sync wave를 5단계로 표준화한 사례를 봤습니다.
- Wave -2: Namespace, NetworkPolicy
- Wave -1: CRD, ConfigMap, Secret
- Wave 0: Deployment, StatefulSet
- Wave 1: Service
- Wave 2: Ingress, PostSync 검증

같은 wave 안에서는 병렬 적용 → 전체 sync 시간 짧게 유지.

## 7. 운영 함정 5선

1. **Hook의 delete-policy 누락**: Job이 누적되어 namespace 오염.
2. **Retry 너무 많이**: 진짜 실패도 30분 후에 알게 됨.
3. **IgnoreDifferences 너무 넓게**: 의도치 않은 drift 감지 실패.
4. **Wave 누락**: CRD 적용 전에 CR 적용 시도 → 실패.
5. **SyncFail hook 누락**: 실패가 사람 눈에 안 들어옴.

## 8. 실습 과제

1. PreSync hook으로 DB migration mock Job 작성.
2. PostSync hook으로 smoke test Job + Slack notification.
3. SyncFail hook으로 PagerDuty webhook.
4. Sync wave 3단계 적용: namespace → deployment → ingress.
5. 일부러 image tag를 잘못 입력해 retry 동작 확인.

## 9. 자가평가 퀴즈

### Q1. Sync wave의 의미는?
1. 리소스 적용 순서 제어
2. UI 색상
3. CPU priority
4. 의미 없음

**정답: 1.**

### Q2. PreSync hook의 대표적 용도는?
1. DB migration
2. UI 렌더링
3. Image build
4. SSO

**정답: 1.**

### Q3. ignoreDifferences에 HPA-managed replicas를 넣지 않으면?
1. ArgoCD가 매번 sync로 replicas를 Git 값으로 되돌려 HPA 무력화
2. 의미 없음
3. UI 깨짐
4. SSO 끊김

**정답: 1.**

### Q4. retry.factor=2의 의미는?
1. 백오프 간격이 매번 2배로 증가
2. 재시도 2회
3. 2초 대기
4. 무관

**정답: 1.**

### Q5. ServerSideApply가 유용한 상황은?
1. 큰 CRD 등에서 client-side apply의 last-applied 한계 우회
2. 모든 경우
3. 무관
4. UI에서만

**정답: 1.**

## 10. 다음 주차

[Week 5: 리포지토리 도구]에서는 Helm·Kustomize·Jsonnet·Config Management Plugin(CMP)을 비교하고, 빅테크가 혼합 사용하는 패턴을 다룹니다.
