---
layout: post
title: "ArgoCD Week 11: Governance, 변경관리, 재해복구 (DR)"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes compliance dr
---

## 학습 목표

- GitOps의 변경관리(Change Management)가 ITIL 관점에서 어떻게 매핑되는지 안다.
- SOC 2 / ISO 27001 변경 통제와 ArgoCD를 연결한다.
- ArgoCD HA 배포와 백업/복원 절차를 설계한다.
- Disaster Recovery: 클러스터 전체 손실 시 복구 시나리오를 시뮬레이션한다.

## 1. ITIL Change Management 매핑

| ITIL 요소 | GitOps/ArgoCD 매핑 |
|----------|--------------------|
| Change Request | Git PR |
| CAB Approval | Required reviewer + branch protection |
| Implementation | ArgoCD auto sync |
| Verification | PostSync hook smoke test + Rollouts analysis |
| Rollback | git revert + auto sync |
| Audit Trail | Git history + ArgoCD audit |

핵심: Git PR이 곧 Change Request. 별도 ITSM 시스템 없이 GitOps만으로 변경 통제 완성.

## 2. SOC 2 CC8.1 (Change Management)

"... entity authorizes, designs, develops, configures, documents, tests, approves, and implements changes ..."

매핑:
- Authorization: GitHub branch protection (required reviewers).
- Documentation: PR description, commit message.
- Testing: PreSync hook, AnalysisTemplate.
- Approval: required code owners.
- Implementation: ArgoCD sync.
- Audit: Git history (immutable when signed commits).

증빙으로 매월 다음 SQL을 자동 생성:

```sql
SELECT pr_number, author, reviewer, merged_at, sync_completed_at
FROM github_argocd_changes
WHERE merged_at BETWEEN '2026-04-01' AND '2026-04-30'
  AND environment = 'prod';
```

## 3. ArgoCD HA 배포

production은 다음 구성:
- argocd-server replicas: 2+
- argocd-application-controller replicas: 2~5 (sharded)
- argocd-repo-server replicas: 3+
- argocd-dex replicas: 2
- Redis HA (sentinel) 또는 외부 Redis cluster
- PodDisruptionBudget: server/repo는 1, controller는 N-1

```yaml
spec:
  controller:
    replicas: 3
    shardingAlgorithm: round-robin
  repo:
    replicas: 5
    autoscale:
      enabled: true
      minReplicas: 5
      maxReplicas: 20
  server:
    replicas: 3
    autoscale:
      enabled: true
```

## 4. 백업

ArgoCD 자체 상태는 Kubernetes etcd에 있습니다:
- Application CR
- AppProject CR
- argocd-secret, argocd-cm, argocd-rbac-cm

백업 대상:
1. Namespace `argocd` 전체 (Velero).
2. argocd-secret(개인키 등): 별도 안전 백업 (Vault).
3. Git 리포: 본래 GitOps Source of Truth → Git 자체 백업.

## 5. 복원 시뮬레이션

**시나리오**: 클러스터 전체 손실, 새 클러스터에 ArgoCD 복원 후 전 앱 sync.

절차:
1. (T+0) 새 클러스터에 ArgoCD 설치 (manifest).
2. (T+5) Velero로 argocd namespace 복원.
3. (T+10) Cluster Secret 재등록 (IAM 권한 회복 후).
4. (T+15) Application controller 자동 reconcile → 모든 앱 sync.
5. (T+30) UI 확인, OutOfSync 없는지.

복원 RTO(Recovery Time Objective): 30~60분 목표.

## 6. DR 패턴 — Active-Passive vs Active-Active

### Active-Passive
- Primary 리전에 ArgoCD, Standby 리전에는 백업만.
- Primary 다운 시 Standby에 ArgoCD 부팅.
- RTO ~30분.

### Active-Active
- 두 리전에 각자 ArgoCD, 같은 Git source 참조.
- 한쪽 다운해도 다른 쪽 무영향.
- 운영 복잡, ApplicationSet으로 destination만 양쪽 분리.
- RTO 0초 (immediate).

빅테크 표준: Tier 0는 Active-Active, 나머지는 Active-Passive.

## 7. 컴플라이언스 자동화 5종

1. **Signed commits 강제**: GitHub branch rule.
2. **Required reviewers**: CODEOWNERS.
3. **변경 ↔ 티켓 연결**: PR 제목에 ticket ID 강제.
4. **변경 evidence 보존**: Git history + ArgoCD audit + S3 WORM.
5. **분기별 access review**: AppProject RBAC 검토 자동 리포트.

## 8. 운영 함정 5선

1. **HA 미적용**: 단일 controller 장애로 전체 sync 멈춤.
2. **백업이 git 의존만**: argocd-secret이 사라지면 dex/SSO 복구 불가.
3. **DR drill 미시행**: 사고 시 첫 시도 = 본방.
4. **Active-Active의 split-brain**: 같은 app을 두 ArgoCD가 동시에 reconcile → 충돌. 적절한 ownership 정의 필요.
5. **컴플라이언스 evidence를 ArgoCD UI에서만 보여줌**: 영구 보존소(S3 등)로 export 필수.

## 9. 빅테크 사례 — Netflix의 GitOps + Spinnaker 공존

Netflix는 long-running Spinnaker 운영 자산이 있어 전체 ArgoCD 전환 대신 새 워크로드부터 ArgoCD로 도입. 두 도구를 같은 Git source 기반으로 운영하며 점진 전환. 큰 조직의 현실적 패턴.

## 10. 실습 과제

1. ArgoCD HA 설치 후 controller 1개를 일부러 죽여 무중단 검증.
2. Velero로 namespace argocd 백업.
3. 새 kind 클러스터에 복원 시뮬레이션.
4. Active-Active를 위한 ApplicationSet 작성 — 두 리전에 동일 앱.
5. SOC 2 CC8.1 매핑 표 자체 작성.

## 11. 자가평가 퀴즈

### Q1. GitOps의 변경관리 매핑에서 PR이 의미하는 것은?
1. ITIL Change Request
2. UI 변경
3. 무관
4. 비용

**정답: 1.**

### Q2. argocd-secret 백업이 git 백업과 별도여야 하는 이유는?
1. dex/SSO·repo 개인키 등 git에 없는 정보가 들어 있어 복구 불가
2. 의미 없음
3. 비용
4. 무관

**정답: 1.**

### Q3. DR drill을 시행 안 하면?
1. 사고 시 첫 시도가 본방 → 실패 확률 큼
2. 안전
3. 비용 절감
4. 무관

**정답: 1.**

### Q4. Active-Active의 잠재 문제는?
1. 두 ArgoCD가 같은 앱을 동시 reconcile → 충돌
2. 더 안전
3. 비용 절감
4. 무관

**정답: 1.**

### Q5. SOC 2 CC8.1의 증빙으로 가장 강력한 것은?
1. Git history + PR 승인 기록 + ArgoCD sync 기록 통합
2. UI 스크린샷
3. 메모
4. 무관

**정답: 1.**

## 12. 다음 주차

[Week 12: 캡스톤]에서는 12주의 모든 컴포넌트를 묶어 가상 빅테크 환경에 GitOps 플랫폼을 풀스택으로 구축합니다.
