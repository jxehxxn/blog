---
layout: post
title: "GoCD Week 6: Pipeline as Code (YAML) — GitOps화"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd pac
---

## 학습 목표

- yaml-config-plugin 설치.
- Config Repo로 pipeline 정의를 git에.
- Pipeline as Code (PaC) 흐름.
- 운영 best practice.

## 1. 비유 — "공장 설계도를 도면실(git)에 두기"

UI 클릭으로 pipeline 만들면 server DB에만. drift, audit 어려움.

Pipeline as Code: pipeline 정의 YAML을 git repo에. GoCD가 polling해 자동 반영. **변경이 PR로 audit**.

## 2. yaml-config-plugin

기본 내장 (v25+). 또는 다음 plugin:
- yaml-config-plugin (공식).
- json-config-plugin.
- groovy-config-plugin.

YAML이 빅테크 표준.

## 3. Config Repo 등록

UI Admin → Config Repositories → Add:
- Type: git.
- URL: `https://github.com/mycorp/gocd-pipelines.git`.
- Plugin: `yaml.config.plugin`.

GoCD가 polling. 변경 시 자동 반영.

## 4. YAML 예시

`pipelines/payments-api.gocd.yaml`:

```yaml
format_version: 10
pipelines:
  payments-api:
    group: payments
    materials:
      git:
        git: https://github.com/mycorp/payments-api.git
        branch: main
    stages:
      - build:
          jobs:
            compile:
              tasks:
                - exec:
                    command: bash
                    arguments: ["-c", "mvn clean install"]
      - test:
          jobs:
            unit:
              tasks:
                - exec: { command: bash, arguments: ["-c", "mvn test"] }
      - deploy:
          approval: manual
          jobs:
            deploy:
              tasks:
                - exec: { command: bash, arguments: ["-c", "./deploy.sh"] }
```

## 5. Pipeline-as-Code 흐름

1. Engineer가 yaml 변경 → PR.
2. Reviewer 검토 → 머지.
3. GoCD가 polling (60s) → config 자동 update.
4. UI에서 새 pipeline 자동 등장.

## 6. Validation

```bash
# 로컬 validation (yaml-config-plugin 도구)
go-yaml-config-plugin --validate pipelines/payments-api.gocd.yaml
```

CI에 통합. 잘못된 YAML 머지 차단.

## 7. Best Practice

- **한 file 한 pipeline**: review 쉬움.
- **shared template**: 반복 제거.
- **branch protection** + reviewer.
- **dry-run** CI.
- **multiple config repos**: 팀별.

## 8. 운영 함정 5선

1. UI에서 변경 후 yaml에 안 반영 → drift.
2. YAML syntax error → pipeline 깨짐.
3. config repo가 server에 push 권한 부족.
4. polling 자주 → git 부담.
5. shared template 없이 복사·붙여넣기.

## 9. 실습 과제

1. `gocd-pipelines` repo 만들기.
2. config repo 등록.
3. 위 YAML로 첫 PaC pipeline.
4. PR → 머지 → 자동 반영 확인.
5. 의도적 syntax error → UI에서 error 확인.

## 10. 자가평가 퀴즈

### Q1. Pipeline as Code 가치?
1. **git PR + audit + 자동 반영**
2. UI 색상 3. 빠름 4. 무관

**정답: 1.**

### Q2. 표준 plugin?
1. **yaml-config-plugin**
2. groovy 3. json 4. UI only

**정답: 1.**

### Q3. Polling 시 무엇이?
1. **config repo의 yaml 변경 감지**
2. material 3. agent 4. 무관

**정답: 1.**

### Q4. Best practice?
1. **한 file 한 pipeline + shared template**
2. 한 file 모두 3. 무관 4. UI 직접

**정답: 1.**

### Q5. drift 원인?
1. **UI에서 변경 후 yaml 안 반영**
2. 안전 3. UI 4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 7: Value Stream Map]에서 GoCD의 시그니처 기능.
