---
layout: post
title: "ArgoCD Week 7: RBAC, SSO, Project — 수천 명 엔지니어의 격리된 셀프서비스"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes security
---

## 학습 목표

- ArgoCD의 SSO 옵션(OIDC, SAML, GitHub, Google)을 안다.
- Dex와 직접 OIDC의 차이를 이해한다.
- RBAC policy CSV 문법을 자유롭게 작성한다.
- AppProject + RBAC + cluster destination 격리로 완전한 멀티테넌트 구조를 설계한다.

## 1. SSO 옵션

ArgoCD는 두 가지 SSO 경로:

### (a) 내장 Dex (브리지)
- SAML, OAuth, LDAP를 OIDC로 변환해 server에 제공.
- argocd-cm에 connectors 설정.

### (b) 직접 OIDC
- 외부 IDP가 이미 OIDC면 Dex 없이 직접 연결 → 단순.

```yaml
# argocd-cm
data:
  url: https://argocd.mycorp.com
  oidc.config: |
    name: Okta
    issuer: https://mycorp.okta.com
    clientID: <client-id>
    clientSecret: $oidc.clientSecret
    requestedScopes: ["openid", "profile", "email", "groups"]
    requestedIDTokenClaims:
      groups:
        essential: true
```

`groups` claim이 핵심 — RBAC에서 그룹 기반 정책으로 직결.

## 2. RBAC Policy CSV

`argocd-rbac-cm`:

```yaml
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:org-admin, applications, *, */*, allow
    p, role:org-admin, clusters, *, *, allow
    p, role:org-admin, repositories, *, *, allow
    p, role:org-admin, projects, *, *, allow

    p, role:dev-readonly, applications, get, */*, allow

    p, proj:team-payments:developer, applications, *, team-payments/*, allow
    p, proj:team-payments:developer, applications, override, team-payments/*, deny

    g, mycorp:platform-admins, role:org-admin
    g, mycorp:payments-engineers, proj:team-payments:developer
```

문법:
- `p, <subject>, <resource>, <action>, <object>, <effect>`
- `g, <group>, <role>` (그룹 → 역할 매핑)

핵심:
- `*`는 모두.
- deny가 allow보다 강함.
- AppProject 범위 정책은 `proj:<project>:<role>` 형식.

## 3. 격리 모델 설계 — 5계명

1. **default project를 사용하지 말 것**. 모든 팀은 자기 project.
2. **sourceRepos 화이트리스트**: 팀이 허용된 git org/repo만 reference 가능.
3. **destinations 화이트리스트**: 자기 namespace + 자기 cluster만.
4. **clusterResourceWhitelist를 비어 두기**: 팀 권한이 cluster-scoped 리소스를 못 만들게 (관리자만 cluster role 부여).
5. **OIDC group 기반**: 사용자 추가/삭제는 IDP에서, ArgoCD는 group만 관리.

## 4. 실제 시나리오 — Payments 팀

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-payments
spec:
  description: Payments team's apps
  sourceRepos:
    - https://github.com/mycorp/payments-*
    - https://charts.mycorp-internal.com/*
  destinations:
    - namespace: payments-*
      server: https://k8s-prod-us-east-1.mycorp.com
    - namespace: payments-*
      server: https://k8s-prod-eu-west-1.mycorp.com
  clusterResourceWhitelist: []
  namespaceResourceWhitelist:
    - group: apps
      kind: '*'
    - group: ''
      kind: '*'
    - group: networking.k8s.io
      kind: '*'
  roles:
    - name: developer
      description: Sync, get, but no override
      policies:
        - p, proj:team-payments:developer, applications, sync, team-payments/*, allow
        - p, proj:team-payments:developer, applications, get, team-payments/*, allow
      groups:
        - mycorp:payments-engineers
    - name: lead
      description: Full app rights within project
      policies:
        - p, proj:team-payments:lead, applications, *, team-payments/*, allow
      groups:
        - mycorp:payments-leads
```

이러면:
- Payments 엔지니어는 payments-* 네임스페이스의 자기 앱만 sync.
- 다른 팀 앱은 안 보임(readonly default 시 get도 불가).
- cluster-scoped 변경 시도 시 거부.

## 5. JWT projects 토큰 — CI 통합

CLI에서 사람이 아닌 머신 토큰:

```bash
argocd proj role create-token team-payments developer \
  --expires-in 24h
```

CI에서 이 토큰으로 `argocd app sync` 호출.

## 6. 빅테크 패턴 — Tier 기반 권한

- Tier 0 (production-critical): lead + 페어 리뷰 필수.
- Tier 1 (production-normal): developer 단독 sync 가능.
- Tier 2 (staging/dev): self-service 광범위.

App label/annotation으로 tier 식별 → RBAC에 분기.

## 7. 운영 함정 5선

1. **default project에 다 몰아넣음**: 결국 격리 실패.
2. **OIDC group claim 미설정**: 그룹 매핑이 안 돌아감.
3. **deny 정책의 우선순위 오해**: allow가 우선이라 가정.
4. **RBAC CSV의 마지막 라인 누락**: YAML escape로 마지막 줄 잘리는 경우 있음.
5. **JWT 토큰의 만료 누락**: 영구 토큰을 CI에 두면 누출 시 회수 어려움.

## 8. 실습 과제

1. Dex로 GitHub SSO 연결.
2. RBAC CSV에 `org-admin`/`team-A-dev`/`team-B-dev` 3 역할 정의.
3. 각 팀 project 만들고 source/dest/resource 격리.
4. JWT 토큰 발급 후 CI mock으로 sync.
5. 다른 팀 app sync 시도해 deny 동작 확인.

## 9. 자가평가 퀴즈

### Q1. Dex의 역할은?
1. SAML/OAuth/LDAP를 OIDC로 브리지
2. RBAC 강제
3. 클러스터 등록
4. UI 렌더

**정답: 1.**

### Q2. RBAC에서 allow와 deny 충돌 시?
1. deny가 우선
2. allow가 우선
3. 무작위
4. 순서대로

**정답: 1.**

### Q3. AppProject `destinations` 화이트리스트의 효과는?
1. 팀이 허용된 cluster + namespace에만 배포 가능
2. UI 색상
3. 의미 없음
4. 비용

**정답: 1.**

### Q4. OIDC `groups` claim이 RBAC에서 중요한 이유는?
1. 그룹 기반 정책으로 사용자 추가/삭제를 IDP에 위임
2. UI에 표시
3. 의미 없음
4. SSO 속도

**정답: 1.**

### Q5. CI에서 사용할 JWT 토큰의 권장 만료는?
1. 짧게(24h 등) + 자동 회전
2. 영구
3. 1년
4. 없음

**정답: 1.**

## 10. 다음 주차

[Week 8: 시크릿 GitOps]에서는 Git에 평문 시크릿을 둘 수 없는 본질 문제를 Vault / SealedSecrets / SOPS / ArgoCD Image Updater로 풀어냅니다.
