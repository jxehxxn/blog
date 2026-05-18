---
layout: post
title: "GoCD Week 8: Manual Approval + Trigger 종류"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd
---

## 학습 목표

- Manual approval 종류.
- Auto trigger 옵션.
- Trigger restriction + 시간.
- Authorization과 결합.

## 1. 비유 — "공장 출하 결재"

자동차 검수 끝 → 영업소 출하 전 매니저 결재. 인쇄, 서명, 기록. GoCD manual approval이 이걸 디지털화.

## 2. Approval 종류

```yaml
stages:
  - name: build
    approval: success    # 기본. 자동 진행
  - name: deploy-stage
    approval:
      type: success     # 명시
  - name: deploy-prod
    approval:
      type: manual      # 사람 결재
      authorization:
        roles: [prod-admin]
```

- **success (auto)**: 이전 stage 성공 시 자동.
- **manual**: 사람이 UI에서 trigger.

## 3. Manual + RBAC

```yaml
approval:
  type: manual
  authorization:
    roles: [prod-deployer]
    users: [alice, bob]
```

role 또는 user 또는 둘 다.

## 4. Trigger 종류

### Auto
material 변경 시 자동.

### Manual
UI/CLI 클릭.

### API
```bash
curl -X POST -u user:token \
  http://gocd.mycorp.com/go/api/pipelines/payments-api/schedule \
  -H "Accept: application/vnd.go.cd+json"
```

CI/CD 외부 시스템 통합.

### Timer (Cron)
```yaml
timer:
  spec: "0 0 2 * * ?"  # daily 2am
  only_on_changes: false
```

정기 build (nightly).

### Pipeline material
upstream 성공.

## 5. Trigger Lock

같은 pipeline 동시 instance 제한:
```yaml
locking: true   # 한 번에 1 instance만
```

stateful 작업에서 중요.

## 6. Conditional 실행

stage 실행 조건 (GoCD는 단순 — Jenkins 같은 풍부한 if 없음). 대안:
- task 안에서 shell 조건.
- 별도 pipeline + dependency.

## 7. 빅테크 패턴

```
[ build (auto) ]
   ↓
[ dev-deploy (auto) ]
   ↓
[ stage-deploy (auto) ]
   ↓
[ prod-deploy (manual + 2 reviewer) ]
```

manual approval에 audit log 자동 (누가 언제 approve).

## 8. 운영 함정 5선

1. **모든 stage manual** → 운영 마비.
2. **모든 stage auto** → prod 사고.
3. **Approval 권한 너무 넓음** (모두 가능).
4. **Timer cron 잘못** (UTC vs local TZ).
5. **Locking 누락 + stateful** → 동시 실행 충돌.

## 9. 실습 과제

1. 4 stage pipeline: build → dev → stage → prod.
2. prod에 manual + role.
3. timer로 nightly stage.
4. locking 활성.
5. 잘못된 사용자 prod trigger → 거부 확인.

## 10. 자가평가 퀴즈

### Q1. Approval type 2종?
1. **success / manual**
2. auto / cron 3. UI 4. 무관

**정답: 1.**

### Q2. Manual + role?
1. **특정 role/user만 trigger**
2. 모두 3. UI 4. 무관

**정답: 1.**

### Q3. API trigger?
1. **외부 시스템 통합**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. Timer cron?
1. **정기 build (nightly 등)**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. Locking?
1. **같은 pipeline 동시 1 instance만**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 9: Elastic Agents]에서 Docker/K8s 자동 agent.
