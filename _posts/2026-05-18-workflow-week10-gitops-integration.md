---
layout: post
title: "Workflow Week 10: GitOps 통합 — Workflow → ArgoCD 완전 자동화"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series gitops
---

## 학습 목표

- CI (Workflow)와 CD (ArgoCD)의 책임 분리.
- 3가지 통합 패턴.
- "Promote between environments" 패턴.
- 안전한 staging → prod 자동 승격.

## 1. 책임 분리

비유: 공장(CI)이 제품을 만들어 창고에 보관. 매장 진열(CD)은 별도 직원이 창고에서 가져옴. 두 직원의 역할이 겹치지 않습니다.

| 책임 | CI (Workflow) | CD (ArgoCD) |
|------|---------------|-------------|
| Source 빌드 | O | X |
| Test/scan | O | X |
| Image 푸시 | O | X |
| Manifest 변경 | O (PR) | X |
| 클러스터 apply | X | O |
| Drift 복구 | X | O |
| Rollback | X (git revert로) | O |

CI가 manifest를 직접 변경 + push까지. apply는 ArgoCD.

## 2. 3가지 통합 패턴

### 패턴 A: Image Updater
ArgoCD Image Updater가 registry watch, 새 image 발견 시 manifest PR 생성.

장점: CI가 manifest 모름. 깔끔.
단점: PR 생성/sync 지연.

### 패턴 B: CI가 직접 PR
CI 마지막 단계에서 manifest repo clone + sed + commit + push.

장점: 즉시 PR.
단점: CI가 git 권한 필요.

### 패턴 C: Webhook trigger
CI 완료 후 ArgoCD API webhook 호출 → sync trigger.

장점: 빠름.
단점: manifest 변경 자체는 별도.

빅테크 일반: 패턴 B + 패턴 A 혼용.

## 3. Promote Between Environments

dev → stage → prod 자동 승격.

```yaml
# manifest repo 구조
apps/myapp/
  base/
    deployment.yaml
  overlays/
    dev/kustomization.yaml         (image tag dev)
    stage/kustomization.yaml       (image tag stage)
    prod/kustomization.yaml        (image tag prod)
```

흐름:
1. CI 빌드 → image push → dev overlay tag 업데이트.
2. ArgoCD dev sync.
3. 자동 smoke test (Argo Workflow PostSync).
4. 성공 → stage overlay tag 업데이트.
5. ArgoCD stage sync.
6. 자동 e2e test.
7. 성공 → prod overlay tag 업데이트 (PR 모드, 사람 승인).
8. merge → ArgoCD prod sync (Canary with Argo Rollouts).

## 4. Promote Workflow 예시

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
spec:
  entrypoint: promote
  templates:
    - name: promote
      steps:
        - - name: update-stage
            template: update-overlay
            arguments: { parameters: [{ name: env, value: stage }] }
        - - name: wait-stage-healthy
            template: wait-argocd-healthy
            arguments: { parameters: [{ name: app, value: myapp-stage }] }
        - - name: e2e-stage
            template: e2e-test
        - - name: pr-prod
            template: create-prod-pr
```

## 5. Argo Rollouts 통합

prod overlay는 단순 Deployment 아닌 Rollout (Argo Rollouts). 자동 canary + Prometheus 기반 abort.

## 6. Notifications

각 단계 결과를 Slack:

```yaml
- name: notify
  inputs:
    parameters: [{ name: message }]
  container:
    image: curlimages/curl
    command: [sh, -c]
    args: ["curl -X POST -d '{\"text\":\"{{inputs.parameters.message}}\"}' $SLACK_WEBHOOK"]
```

## 7. 보안 — CI의 git 권한

CI가 manifest repo에 push하려면 token. 위험.

대책:
- 별도 deploy bot 계정 (전용 PAT).
- 권한 최소화 (특정 repo만).
- token Vault 등 secret manager에 저장.
- 또는 ArgoCD Image Updater로 CI에 git 권한 0.

## 8. Rollback

- ArgoCD rollback: 옛 commit의 manifest로 sync.
- 또는 git revert → 자동 sync.
- 또는 Argo Rollouts abort.

빅테크 표준: git revert → 모든 audit 명확.

## 9. 운영 함정 5선

1. CI가 prod 직접 apply → 누가 무엇을 했는지 추적 어려움.
2. PR 자동 머지 → 사람 승인 부재 → 사고.
3. Image tag latest → 어느 이미지가 sync됐는지 모름.
4. CI 토큰 권한 과다.
5. e2e test 없이 prod 승격.

## 10. 빅테크 사례 — Spotify

Spotify Backstage + Argo Workflows + ArgoCD가 통합. 개발자가 push → 자동 dev → 사람 승인 → stage → 자동 prod canary.

## 11. 실습

```bash
# 1. 3 환경 overlay 구조
# 2. CI 마지막 step에서 dev overlay 업데이트 PR
# 3. ArgoCD dev sync 자동
# 4. PostSync로 smoke test → 결과 따라 stage 자동 promote
# 5. prod는 사람 승인 게이트
```

## 12. 자가평가 퀴즈

### Q1. CI/CD 책임 분리?
1. **CI는 build/test/manifest 변경, CD는 cluster apply/drift/rollback**
2. CI가 모두
3. CD가 모두
4. 무관

**정답: 1.**

### Q2. Image Updater 패턴의 장점?
1. **CI가 manifest를 몰라도 됨 + 깔끔**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q3. promote between env의 핵심?
1. **단계별 자동 게이트 + 사람 승인은 prod에만**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q4. Rollback의 빅테크 표준?
1. **git revert → audit 명확**
2. CI 재실행
3. UI
4. 무관

**정답: 1.**

### Q5. CI git 토큰 권한 권장?
1. **최소화 + Vault 저장 + 전용 bot 계정**
2. admin
3. UI
4. 무관

**정답: 1.**

## 13. 다음 주차

[Week 11: 보안]에서 secrets, sandbox, supply chain을 다룹니다.
