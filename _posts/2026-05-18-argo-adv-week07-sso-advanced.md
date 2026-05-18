---
layout: post
title: "ArgoCD 심화 Week 7: SSO Advanced — OIDC, SAML, Dex Connector"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced sso platform senior-series
---

## 학습 목표

- Dex connector 종류.
- group claim mapping.
- JWT token 발급 정책.
- 외부 IDP 통합 (Okta, Azure AD).

## 1. Dex Connector

- OAuth: GitHub, Google, GitLab.
- OIDC: 표준.
- SAML.
- LDAP.
- LinkedIn.

## 2. Okta OIDC

```yaml
data:
  dex.config: |
    connectors:
      - type: oidc
        id: okta
        name: Okta
        config:
          issuer: https://mycorp.okta.com
          clientID: $okta-client-id
          clientSecret: $okta-secret
          insecureEnableGroups: true
```

## 3. Azure AD

```yaml
connectors:
  - type: microsoft
    id: azuread
    name: Azure AD
    config:
      clientID: $azure-client-id
      clientSecret: $azure-secret
      tenant: <tenant-id>
      groups: ["argocd-admins", "argocd-developers"]
```

## 4. Group Claim Mapping

OIDC가 보내는 group claim → RBAC.

```yaml
oidc.config: |
  ...
  requestedIDTokenClaims:
    groups:
      essential: true
```

RBAC:
```yaml
policy.csv: |
  g, mycorp:platform-admins, role:org-admin
  g, mycorp:payments-engineers, proj:team-payments:developer
```

## 5. JWT Token

CI 용:
```bash
argocd proj role create-token team-payments developer \
  --expires-in 24h
```

24h 만료 권장. Vault에 저장. 자동 회전.

## 6. ArgoCD Token vs SSO

- SSO: 사람 (browser/CLI).
- Project JWT: 자동화 (CI).

분리 운영.

## 7. 운영 함정

1. SSO secret 노출.
2. group claim 누락 → RBAC 안 됨.
3. JWT 무기한 → 누출 시 회수 어려움.
4. session timeout 너무 길게.
5. dex 단일 instance.

## 8. 자가평가

### Q1. Dex의 역할?
1. **SSO bridge (OAuth/SAML → OIDC)** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. group claim 가치?
1. **그룹 기반 RBAC → 사용자 추가/삭제는 IDP** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. CI 용 권장?
1. **Project JWT 짧은 만료** 2. SSO 3. UI 4. 무관

**정답: 1.**

### Q4. Azure AD 통합?
1. **Microsoft connector** 2. SAML만 3. UI 4. 무관

**정답: 1.**

### Q5. SSO secret 운영?
1. **K8s Secret + Vault 등** 2. 평문 3. UI 4. 무관

**정답: 1.**

## 9. 다음

[Week 8: ArgoCD Operator multi-tenancy].
