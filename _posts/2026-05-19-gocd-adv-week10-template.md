---
layout: post
title: "GoCD 심화 Week 10: YAML Template + Governance"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd governance senior
---

## 학습 목표
- Pipeline template.
- 사내 표준 강제.
- Multi-team governance.

## 1. Pipeline Template (내장)

```yaml
templates:
  microservice-template:
    stages:
      - build:
          jobs:
            compile:
              tasks: [...]
      - test:
          jobs: [...]
      - deploy-stage:
          jobs: [...]
      - deploy-prod:
          approval: manual
          jobs: [...]

pipelines:
  payments-api:
    group: payments
    template: microservice-template
    materials: { git: ... }
```

20개 service가 같은 template → 표준화.

## 2. Parameters

```yaml
template:
  microservice-template:
    params:
      - DEPLOY_REGION
    stages: [...]

pipelines:
  payments-api:
    template: microservice-template
    parameters:
      DEPLOY_REGION: us-east-1
```

## 3. External Template Engine

복잡 case: Jinja/Cookiecutter로 yaml 생성. CI에 통합.

## 4. Governance Policy

빅테크 사내 규칙:
- 모든 pipeline은 template 사용 (no exception).
- prod stage는 반드시 manual approval.
- secret은 secret plugin만 (env 평문 금지).
- artifact retention 30d 이상.

이를 plugin 또는 외부 lint로 강제.

## 5. Yaml Lint

```bash
go-yaml-config-plugin --validate --policy=mycorp-policy.json my-pipeline.yaml
```

PR CI에서 머지 차단.

## 6. Pipeline Group 정책

- 신규 group은 platform team approval.
- Template 사용률 100%.

## 7. Audit Dashboard

각 pipeline의 template 사용률, manual approval 사용, secret usage 통계.

## 8. 자가평가
### Q1. Pipeline Template 가치? **표준화**. 정답 1.
### Q2. Parameters? **template 인스턴스화**. 정답 1.
### Q3. Governance? **template + lint + audit**. 정답 1.
### Q4. lint 적용? **PR CI**. 정답 1.
### Q5. 사내 정책 예? **manual approval 강제**. 정답 1.
