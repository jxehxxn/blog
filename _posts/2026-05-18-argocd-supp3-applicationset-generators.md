---
layout: post
title: "ArgoCD 보충 3: ApplicationSet generator 8종 전수 분석"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes supplement
---

3주차에서 사용 시점만 짚은 generator 8종을 각각 깊게 다룹니다.

## 1. List Generator

```yaml
generators:
  - list:
      elements:
        - cluster: dev
          url: https://k8s-dev.mycorp.com
          replicas: "1"
        - cluster: prod
          url: https://k8s-prod.mycorp.com
          replicas: "3"
```

- 가장 단순. 정적 매개변수.
- 사용처: 적은 수의 환경/클러스터.
- 함정: 새 클러스터 추가 시 yaml 직접 수정 필요.

## 2. Cluster Generator

```yaml
generators:
  - clusters:
      selector:
        matchLabels:
          env: prod
```

- ArgoCD에 등록된 클러스터 중 selector 매칭.
- 새 클러스터 추가 시 자동 인지.
- 함정: selector 누락 시 모든 클러스터 fan-out.

label 추가 시 클러스터 secret에:

```yaml
metadata:
  labels:
    env: prod
    region: us-east-1
```

## 3. Git Generator — Files

```yaml
generators:
  - git:
      repoURL: https://github.com/mycorp/cluster-config.git
      revision: HEAD
      files:
        - path: "clusters/**/config.json"
```

- 지정 경로의 JSON/YAML 파일을 매개변수로.
- 각 파일이 1개 Application 매개변수 셋.
- 사용처: 인프라가 다른 git에 있을 때.

## 4. Git Generator — Directories

```yaml
generators:
  - git:
      repoURL: https://github.com/mycorp/manifests.git
      revision: HEAD
      directories:
        - path: "apps/*"
        - path: "apps/excluded"
          exclude: true
```

- 디렉토리 목록을 generator로.
- `exclude: true`로 특정 디렉토리 제외.
- 함정: deep glob 사용 시 성능 저하.

## 5. Matrix Generator

```yaml
generators:
  - matrix:
      generators:
        - clusters:
            selector:
              matchLabels:
                env: prod
        - git:
            directories:
              - path: "apps/*"
```

- 두 generator의 카테시안 곱.
- 클러스터 N × 앱 M = N×M Application.
- 함정: 폭발 — 매개변수 범위 신중. 4×100×50 = 20,000개 됨.

## 6. Merge Generator

```yaml
generators:
  - merge:
      mergeKeys:
        - cluster
      generators:
        - clusters: {}
        - list:
            elements:
              - cluster: prod
                tier: "0"
              - cluster: dev
                tier: "2"
```

- merge key로 join. 클러스터별 메타데이터 추가.
- 사용처: 클러스터마다 다른 설정을 별도 list로 관리.

## 7. SCM Provider Generator

```yaml
generators:
  - scmProvider:
      github:
        organization: mycorp
        tokenRef:
          secretName: github-token
          key: token
      filters:
        - repositoryMatch: ^service-
        - branchMatch: ^(main|master)$
      cloneProtocol: ssh
```

- GitHub Org / GitLab Group의 repo 자동 발견.
- 신규 repo가 생기면 자동 Application 생성.
- 함정: org admin 권한 X — Read만.

## 8. Pull Request Generator

```yaml
generators:
  - pullRequest:
      github:
        owner: mycorp
        repo: webapp
        tokenRef:
          secretName: gh-token
          key: token
        labels:
          - preview
      requeueAfterSeconds: 60
```

- 열린 PR마다 Preview Application.
- PR 닫히면 자동 삭제.
- 함정: preserveResourcesOnDeletion=false 권장. 메타데이터 정책 합의 필요.

## 9. Cluster Decision Resource Generator

```yaml
generators:
  - clusterDecisionResource:
      configMapRef: my-configmap
      name: my-decision
      requeueAfterSeconds: 180
```

- 외부 controller(예: KubeFed, Open Cluster Management)가 결정한 클러스터 목록을 인입.
- 다중 클러스터 메타-관리 시스템과 연계.

## 10. Generator 조합 best practice

- 단순 환경: List.
- 다클러스터 fan-out: Cluster + selector.
- 동적 추가: SCM Provider.
- PR preview: Pull Request.
- 매개변수 풍부: Matrix(주의).

## 11. 실전 함정 정리

1. Matrix 폭발.
2. SCM token 과다 권한.
3. PullRequest cleanup 누락.
4. Cluster selector 미설정.
5. Git generator의 glob 성능.
6. Template name 충돌 (중복 Application name).

## 12. 결론

8종 generator를 적재적소에 쓰면 200명·1,000 앱 환경도 yaml 100줄 이내로 관리. 그 정도가 GitOps의 진짜 힘.
