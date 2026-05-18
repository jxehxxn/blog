---
layout: post
title: "GoCD Week 4: Materials — Git, Multi-material, Polling"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd
---

## 학습 목표

- Material 종류와 GoCD의 polling 모델.
- Multi-material로 monorepo 또는 multi-repo 통합.
- Pipeline material (다른 pipeline의 산출물).
- Webhook 통합.

## 1. 비유 — "공장 입구 검문소"

Material = 공장에 들어오는 원자재 출처. Git repo, 다른 pipeline 산출물, package repo 등. 검문소(server)가 주기적으로 출처를 polling, 변경 시 pipeline trigger.

## 2. Material 종류

| 종류 | 사용 |
|------|------|
| **git** | 가장 흔함. branch 지정 |
| mercurial | hg |
| svn | legacy |
| perforce (p4) | 게임/금융 일부 |
| tfs | Microsoft |
| package | apt/yum repo |
| **pipeline** | 다른 pipeline의 성공 trigger |

## 3. Git Material 예

```yaml
materials:
  - git:
      url: https://github.com/mycorp/myapp.git
      branch: main
      shallow_clone: true
      destination: src
      auto_update: true   # polling on
      filter:
        ignore: ["**/*.md"]
```

`filter.ignore`: 문서만 변경된 commit은 trigger 안 함.

## 4. Polling vs Webhook

기본은 polling (60초). 효율과 즉시성을 위해 webhook 권장:

```
GitHub webhook URL: http://gocd.mycorp.com/go/api/webhooks/github/notify
```

GitHub push → 즉시 GoCD trigger.

빅테크 표준: polling + webhook 병행 (fallback).

## 5. Multi-material

```yaml
materials:
  - git:
      url: https://github.com/mycorp/app.git
      destination: app
  - git:
      url: https://github.com/mycorp/config.git
      destination: config
```

둘 다 polling. 한쪽이라도 변경 → trigger. 두 변경이 같은 시점이면 묶어서 한 번 trigger (fan-in).

## 6. Pipeline Material (Dependency)

다른 pipeline의 성공이 trigger:

```yaml
materials:
  - pipeline:
      pipeline: upstream
      stage: package
```

upstream pipeline의 package stage 성공 → 이 pipeline 자동 trigger. **dependency graph 핵심**. 7주차 VSM에서 시각화.

## 7. Package Material

apt/yum/Maven/npm 등 외부 package:

```yaml
materials:
  - package:
      name: my-package
      repository: my-repo
```

upstream package가 update → trigger.

## 8. 운영 함정 5선

1. **Polling 너무 자주**: git server 부담.
2. **Webhook secret 누락**: 외부 누구나 trigger.
3. **shallow_clone false + 큰 repo**: 매번 full clone 느림.
4. **Filter ignore 누락**: 문서 commit도 build.
5. **Multi-material destination 충돌**: 같은 dir에 다른 source.

## 9. 빅테크 패턴 — Source of Truth 분리

- App code: 한 repo.
- K8s manifest: 다른 repo (GitOps).
- Pipeline 정의: 또 다른 repo (PaC).

Multi-material로 모두 한 pipeline에 통합 가능.

## 10. 실습 과제

1. Git material로 github repo 등록.
2. webhook 설정 → GitHub push 즉시 trigger 확인.
3. 두 번째 git material 추가 (config repo) → fan-in 동작 확인.
4. Filter ignore로 `*.md` 제외.
5. upstream pipeline 만들고 downstream에 pipeline material로 연결.

## 11. 자가평가 퀴즈

### Q1. Material 종류 가장 흔한 것?
1. **git**
2. svn 3. mercurial 4. p4

**정답: 1.**

### Q2. Polling 기본 간격?
1. **60초**
2. 5초 3. 1시간 4. 무관

**정답: 1.**

### Q3. Webhook URL pattern?
1. **/go/api/webhooks/github/notify**
2. /webhook 3. /trigger 4. 무관

**정답: 1.**

### Q4. Pipeline material 의미?
1. **다른 pipeline의 성공이 trigger → dependency**
2. 같은 pipeline
3. UI
4. 무관

**정답: 1.**

### Q5. Filter ignore의 가치?
1. **문서 등 무관 변경은 trigger 안 함**
2. 빠름 3. UI 4. 무관

**정답: 1.**

## 12. 다음 주차

[Week 5: Environment + Pipeline Group]에서 namespace/folder 격리를 다룹니다.
