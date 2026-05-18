---
layout: post
title: "Policy Week 11: GitOps + Policy Lifecycle"
date: 2026-05-18 20:00:00 +0900
categories: policy opa kyverno gitops platform senior-series
---

## 학습 목표

- 정책 자체를 GitOps로 관리.
- 정책 라이프사이클 (draft → dryrun → warn → enforce).
- 정책 변경 audit + rollback.
- 다중 cluster 배포.

## 1. Policy as Code in Git

ConstraintTemplate, Constraint, ClusterPolicy 모두 K8s CRD → Git에 두고 ArgoCD/Flux로 sync.

```
policies/
  baseline/
    require-labels.yaml
    no-privileged.yaml
  payments/
    require-team-label.yaml
```

ArgoCD ApplicationSet으로 cluster fan-out.

## 2. 라이프사이클

```
draft (PR) → dryrun (cluster) → warn → enforce (deny)
```

각 단계 시간 (예: 1주씩).

## 3. PR Workflow

1. 새 정책 PR (draft).
2. CI: opa test / kyverno test.
3. dryrun enforcement로 머지.
4. 1주 observation (violations 분석).
5. warn → 사용자에게 알람.
6. enforce.

## 4. 정책 변경 Audit

git history. 모든 변경의 author/reviewer/diff.

```bash
git log --all --oneline -- policies/
```

SOC 2 CC8.1 evidence.

## 5. Rollback

policy로 인한 production 차단 → `git revert` → ArgoCD 자동 sync.

수동 disable:
```bash
kubectl patch constraint require-labels --type merge -p '{"spec":{"enforcementAction":"dryrun"}}'
```

## 6. Multi-cluster Policy

ApplicationSet으로 모든 cluster:
```yaml
spec:
  generators:
    - clusters:
        selector:
          matchLabels: { policy-tier: tier1 }
  template:
    spec:
      source:
        path: policies/baseline
      destination:
        server: '{{server}}'
        namespace: gatekeeper-system
```

cluster label로 tier 별 정책 차등.

## 7. Policy Catalog

사내 wiki/Backstage:
- 정책 이름, 설명, owner.
- 적용 cluster.
- exempt 목록.
- 마지막 변경.

신규 엔지니어 onboarding 효율.

## 8. Test in PR

```yaml
# .github/workflows/policy-test.yaml
- run: opa test policies/
- run: kyverno test policies/
- run: conftest test test-manifests/ --policy policies/
```

머지 전 검증.

## 9. 운영 함정

1. PR test 없음 → 잘못된 policy로 cluster 차단.
2. dryrun 단계 없이 enforce → 운영 사고.
3. Rollback 절차 모름.
4. catalog 부재 → 어떤 정책 있는지 모름.
5. multi-cluster drift.

## 10. 자가평가 퀴즈

### Q1. Policy as Code GitOps 방식?
1. **CRD를 Git에 두고 ArgoCD sync**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. 라이프사이클?
1. **draft → dryrun → warn → enforce**
2. 한 번에 enforce 3. UI 4. 무관

**정답: 1.**

### Q3. Rollback 방법?
1. **git revert 또는 enforcementAction dryrun**
2. 수동 삭제 3. UI 4. 무관

**정답: 1.**

### Q4. Multi-cluster 배포?
1. **ApplicationSet + cluster generator**
2. 수동 3. UI 4. 무관

**정답: 1.**

### Q5. PR test의 가치?
1. **잘못된 policy로 차단 사고 방지**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 11. 다음

[Week 12: 캡스톤].
