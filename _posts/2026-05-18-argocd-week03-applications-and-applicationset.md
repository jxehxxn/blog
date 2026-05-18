---
layout: post
title: "ArgoCD Week 3: Application과 ApplicationSet — 1,000개 앱을 선언적으로 다루기"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes
---

## 학습 목표

- 단일 Application의 한계를 이해한다.
- ApplicationSet의 핵심 개념 — generator + template을 이해한다.
- 8종 generator(List, Cluster, Git, Matrix, Merge, SCM Provider, Pull Request, Cluster Decision)의 사용 시점을 안다.
- App-of-Apps 패턴과 ApplicationSet의 트레이드오프를 비교한다.

## 1. 왜 ApplicationSet인가

200개 마이크로서비스 × 3 환경(dev/stage/prod) × 5 리전 = 3,000개 Application. 사람이 직접 yaml을 3,000개 만들 수 없습니다. 두 가지 해법:

1. **App-of-Apps**: 부모 Application이 자식 Application들의 manifest를 포함. 한 번 set up하면 자식이 자동 생성.
2. **ApplicationSet**: 선언적 generator로 N개 Application을 동적 생성.

빅테크 표준은 ApplicationSet입니다. 이유: drift 추적 가능, generator 재사용, RBAC와 결합.

## 2. ApplicationSet 기본 구조

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: dev
            url: https://kubernetes.dev.mycorp.com
          - cluster: prod
            url: https://kubernetes.prod.mycorp.com
  template:
    metadata:
      name: '{{cluster}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/mycorp/manifests.git
        targetRevision: HEAD
        path: 'apps/guestbook/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: guestbook
      syncPolicy:
        automated: { prune: true, selfHeal: true }
```

핵심:
- `generators`가 매개변수 N세트를 만든다.
- `template`이 그 매개변수를 받아 N개 Application을 찍는다.

## 3. 8종 Generator — 사용 시점

### (a) List
- 사용: 정적인 클러스터/env 목록.
- 예: dev/stage/prod 3개 환경 정의.

### (b) Cluster
- ArgoCD에 등록된 모든(또는 라벨 매칭) 클러스터로 fan-out.

```yaml
generators:
  - clusters:
      selector:
        matchLabels:
          env: prod
```

빅테크 멀티리전 fan-out의 표준.

### (c) Git (files)
- Git 리포 안의 JSON/YAML 파일들을 매개변수 소스로.

```yaml
generators:
  - git:
      repoURL: https://github.com/mycorp/cluster-config.git
      revision: HEAD
      files:
        - path: "clusters/*/config.json"
```

### (d) Git (directories)
- Git 리포의 디렉토리 목록을 generator로.

```yaml
generators:
  - git:
      repoURL: https://github.com/mycorp/manifests.git
      revision: HEAD
      directories:
        - path: "apps/*"
```

`apps/` 아래 디렉토리 N개 → Application N개 자동 생성.

### (e) Matrix
- Generator 2개를 카테시안 곱. 예: 환경 3개 × 앱 50개 = 150 Application.

```yaml
generators:
  - matrix:
      generators:
        - list:
            elements: [{env: dev}, {env: prod}]
        - git:
            directories: [{path: "apps/*"}]
```

### (f) Merge
- Generator 결과를 join.

### (g) SCM Provider
- GitHub Org, GitLab Group의 리포 목록을 generator로.

```yaml
generators:
  - scmProvider:
      github:
        organization: mycorp
        tokenRef: { secretName: github-token, key: token }
      filters:
        - repositoryMatch: ^service-
```

`service-*` 리포가 새로 생기면 자동으로 Application 등록.

### (h) Pull Request
- 열려 있는 PR마다 Preview Application 자동 생성.

```yaml
generators:
  - pullRequest:
      github:
        owner: mycorp
        repo: webapp
        tokenRef: { secretName: gh-token, key: token }
```

PR이 merge·close되면 Preview Application 자동 삭제 (`syncPolicy.preserveResourcesOnDeletion=false` 권장).

세부 행동·예시는 [보충 3: ApplicationSet generator 8종 전수 분석] 참고.

## 4. App-of-Apps vs ApplicationSet — 결정 가이드

| 항목 | App-of-Apps | ApplicationSet |
|------|-------------|----------------|
| 자식 생성 방식 | YAML manifest 작성 | generator로 동적 |
| Drift 추적 | child Application 자체가 추적 | controller가 직접 관리 |
| 동적 환경 (PR/cluster) | 어려움 | 쉬움 |
| 초기 학습 | 쉬움 | 중간 |
| 빅테크 권장 | 레거시 | **현행 표준** |

## 5. 빅테크 사례 — Intuit의 Matrix + SCM 패턴

Intuit는 GitHub Org에 새 서비스가 추가되면 SCM Provider generator가 자동 인지 → Matrix로 dev/stage/prod 환경 × 5리전 = 15 Application 자동 생성. 사람이 yaml을 만지는 시점이 거의 없습니다.

## 6. 흔한 함정 5선

1. **PullRequest generator의 정리 누락**: PR이 머지돼도 preview env가 남아 클러스터 자원 누수.
2. **Matrix 폭발**: 3 × 3 × 3 = 27이 아니라 30 × 30 × 30 = 27,000. 매개변수 범위 신중.
3. **SCM Provider의 토큰 권한 과다**: org admin 주지 말 것 (Read만).
4. **Cluster generator의 selector 누락**: 모든 클러스터에 fan-out돼 의도치 않은 production 배포.
5. **Template의 `name` 충돌**: cluster 이름 + app 이름 결합으로 고유성 보장.

## 7. 실습 과제

1. List generator로 dev/prod 2 환경 ApplicationSet 작성.
2. Matrix로 환경 × 3개 앱 = 6 Application 생성.
3. PullRequest generator를 자신의 GitHub 리포에 적용해 preview env 만들기.
4. 의도적으로 PR을 닫고 preview env가 30초 안에 사라지는지 확인.
5. 한 페이지 보고서로 generator 별 사용 시점·주의점 정리.

## 8. 자가평가 퀴즈

### Q1. ApplicationSet이 App-of-Apps보다 권장되는 이유는?
1. 선언적 generator로 동적 환경 대응 + drift 추적
2. UI가 더 예뻐서
3. 더 빠름
4. 무관

**정답: 1.**

### Q2. Matrix generator 사용 시 가장 큰 위험은?
1. 매개변수 폭발로 클러스터 자원 폭증
2. UI 느려짐
3. SSO 끊김
4. Redis 만료

**정답: 1.**

### Q3. PullRequest generator를 쓸 때 잊으면 안 되는 것은?
1. PR 종료 시 Application 자동 삭제 정책
2. UI 색상
3. CPU limit
4. 의미 없음

**정답: 1.**

### Q4. SCM Provider generator의 적절한 토큰 권한은?
1. Read 권한만
2. Org admin
3. Owner
4. 권한 없음

**정답: 1.**

### Q5. Cluster generator의 selector를 비워두면?
1. 모든 클러스터에 fan-out — 의도치 않은 production 배포 위험
2. 0개 클러스터
3. 무작위 1개
4. dev만

**정답: 1.**

## 9. 다음 주차

[Week 4: Sync 전략]에서는 sync wave/hook/retry/ignore-differences를 활용해 "안전한 변경 관리"를 다룹니다.
