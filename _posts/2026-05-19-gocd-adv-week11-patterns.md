---
layout: post
title: "GoCD 심화 Week 11: Pipeline Pattern Catalog"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd patterns senior
---

## 학습 목표
- Microservice pattern.
- Monorepo pattern.
- ML/data pipeline pattern.
- Blue-green / canary.

## 1. Microservice Pattern

각 service = 1 pipeline. template 표준.
- build → test → image push → multi-region deploy.

## 2. Monorepo Pattern

1 git repo + N service. material filter:
```yaml
materials:
  - git:
      url: https://github.com/mycorp/monorepo.git
      filter:
        whitelist: ["services/payments/**"]
```

각 service 별 pipeline. fan-out.

## 3. ML Pipeline

- data preparation.
- training.
- evaluation.
- model registry push.
- canary deploy.

stages가 길고 stateful. artifact 크.

## 4. Blue-Green

```yaml
stages:
  - deploy-green:
      jobs: [...]
  - smoke-test:
      jobs: [...]
  - switch-to-green:
      approval: manual
      jobs: [...]   # traffic 전환
```

## 5. Canary

```yaml
stages:
  - deploy-canary:
      jobs: [{ tasks: [{exec: "deploy 5%"}]}]
  - monitor:
      jobs: [{ tasks: [{exec: "monitor 10min"}]}]
  - rollout-full:
      approval: manual
      jobs: [...]
```

monitor stage에서 metric query 후 fail/success.

## 6. DB Migration

```yaml
stages:
  - migrate:
      jobs:
        run-migration:
          tasks: [{exec: "flyway migrate"}]
  - deploy:
      jobs: [...]
```

migration이 stage로.

## 7. 자가평가
### Q1. monorepo? **material filter + 여러 pipeline**. 정답 1.
### Q2. ML pipeline? **prep/train/eval/registry/deploy**. 정답 1.
### Q3. Blue-Green? **두 환경 + traffic 전환**. 정답 1.
### Q4. Canary monitor? **metric query stage**. 정답 1.
### Q5. DB migration? **별도 stage 우선**. 정답 1.
