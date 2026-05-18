---
layout: post
title: "GoCD 심화 Week 6: ChatOps + IDP 통합"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd chatops idp senior
---

## 학습 목표
- Slack bot 통합.
- LDAP/SAML SSO.
- Backstage / 사내 IDP.

## 1. Slack

Notification plugin (이전 코스). 추가:
- Build approval Slack에서 직접.
- Deployment status command.
- `/gocd trigger payments-api` 같은 slash command.

## 2. LDAP/SAML SSO

```yaml
auth_config:
  - id: ldap
    plugin_id: cd.go.authentication.ldap
    properties:
      Url: ldap://ldap.mycorp.com:389
      ManagerDN: cn=gocd,ou=svc,dc=mycorp,dc=com
      ManagerPassword: $ldap_pass
      SearchBase: ou=people,dc=mycorp,dc=com
      LoginFilter: (uid={0})
```

SAML/OAuth 비슷한 plugin.

## 3. Role Mapping

```yaml
role_mapping:
  - role_name: gocd-admin
    ldap_group: cn=devops,ou=groups,dc=mycorp
```

사내 LDAP 그룹 ↔ GoCD role.

## 4. Backstage 통합

Backstage plugin이 GoCD API 호출.
- Service catalog의 "Deploy" 버튼.
- pipeline status panel.

## 5. ServiceNow / Jira

Notification webhook으로:
- prod deploy 시 자동 ticket.
- ticket ID를 commit message에 강제.

## 6. 운영 함정
1. SSO secret 노출.
2. LDAP latency → login 느림.
3. role mapping 자동화 부재.
4. ChatOps 권한 검증 누락.

## 7. 자가평가
### Q1. SSO plugin? **LDAP/SAML/OAuth**. 정답 1.
### Q2. role mapping? **LDAP group ↔ GoCD role**. 정답 1.
### Q3. ChatOps? **Slack slash command**. 정답 1.
### Q4. Backstage? **service catalog deploy 버튼**. 정답 1.
### Q5. ServiceNow? **자동 ticket**. 정답 1.
