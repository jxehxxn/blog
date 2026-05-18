---
layout: post
title: "GoCD Week 10: Plugins + Secret Management"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd plugins secrets
---

## 학습 목표

- Plugin 카테고리 5종.
- 자주 쓰는 plugin.
- Secret management (Vault, AWS SM).

## 1. Plugin 카테고리

1. **SCM**: git/svn/p4 외 다른 SCM 통합.
2. **Task**: ant, docker, helm 등.
3. **Notification**: Slack, Email.
4. **Authentication**: LDAP, OAuth.
5. **Authorization**: role 외부 매핑.
6. **Elastic Agent**: Docker/K8s.
7. **Secret**: Vault/AWS SM/CyberArk.
8. **Config Repo**: yaml/json/groovy.
9. **Artifact**: Docker registry, Maven.

## 2. Plugin 설치

UI Admin → Plugins. 또는 jar 파일을 `plugins/external/` 폴더에.

빅테크: 사내 mirror + 자동 deployment.

## 3. Secret Management

전통: pipeline variable에 평문. **금지**.

권장:
- HashiCorp Vault plugin.
- AWS Secrets Manager plugin.
- File-based plugin.

## 4. Vault 통합

```yaml
secret_configs:
  - id: vault
    plugin_id: cd.go.secrets.vault
    properties:
      VaultUrl: https://vault.mycorp.com
      AuthMethod: token
      Token: $vault_token
      SecretEngine: secret
```

사용:
```yaml
environment_variables:
  DB_PASSWORD: "{{SECRET:[vault][payments/db][password]}}"
```

GoCD가 runtime에 fetch.

## 5. AWS Secrets Manager

```yaml
secret_configs:
  - id: aws-sm
    plugin_id: cd.go.secrets.aws
    properties:
      Region: us-east-1
      ...
```

## 6. Slack Notification

```yaml
notification:
  plugin_id: cd.go.contrib.notification.slack
  webhook: https://hooks.slack.com/...
  channel: "#cd-alerts"
  on_failure: true
  on_success: false
```

PostSync 알림.

## 7. 사내 자체 Plugin

Java/Groovy로 자체 plugin 가능. ThoughtWorks API.

빅테크 일부가 사내 도구 통합 plugin 작성.

## 8. 운영 함정 5선

1. **Plugin 업데이트 미관리**: 보안 패치 누락.
2. **Secret pipeline variable에 평문**.
3. **Vault token 자체 노출**.
4. **Plugin compatibility** (server 버전 ↔ plugin).
5. **Notification 폭주** (모든 build 알람).

## 9. 실습 과제

1. Slack notification plugin 설정.
2. Vault dev mode + secret plugin.
3. Pipeline variable로 vault secret 참조.
4. 의도적 fail → Slack 알람 확인.
5. Vault에서 secret 변경 → 다음 build에 반영.

## 10. 자가평가 퀴즈

### Q1. Plugin 카테고리에 포함되지 않는 것?
1. **DB plugin** (없음)
2. SCM 3. Task 4. Notification

**정답: 1.**

### Q2. Secret 평문 변수 위험?
1. **모든 log/UI에서 노출**
2. UI만 3. 안전 4. 무관

**정답: 1.**

### Q3. Vault 통합 방법?
1. **secret plugin + {{SECRET:...}}**
2. UI 입력 3. file 4. 무관

**정답: 1.**

### Q4. Notification 권장?
1. **on_failure만 (또는 일부 success)**
2. 모두 3. 무 4. UI

**정답: 1.**

### Q5. 사내 plugin 작성?
1. **Java/Groovy 가능**
2. 불가 3. UI 4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 11: GoCD vs 다른 도구]에서 의사결정 기준.
