---
layout: post
title: "ArgoCD 심화 Week 9: ApplicationSet 조합 패턴"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced platform senior-series
---

## 학습 목표

- Matrix + Plugin 결합.
- Multi-source ApplicationSet.
- 환경별 정밀 제어.

## 1. Matrix + Plugin

cluster 목록은 plugin (CMDB), 앱 목록은 git directory:

```yaml
generators:
  - matrix:
      generators:
        - plugin:
            configMapRef: { name: cmdb-cluster }
        - git:
            directories: [{path: "apps/*"}]
```

각 cluster × app 조합.

## 2. Merge

```yaml
generators:
  - merge:
      mergeKeys: [cluster]
      generators:
        - clusters: {}
        - list:
            elements:
              - { cluster: prod, tier: "0" }
              - { cluster: dev, tier: "2" }
```

cluster metadata + 환경 정보 join.

## 3. Multi-source App

ApplicationSet template에서:
```yaml
template:
  spec:
    sources:
      - repoURL: chart-museum
        chart: payments
        helm:
          valueFiles: [$values/{{cluster}}/values.yaml]
      - repoURL: values-repo
        ref: values
```

chart는 vendor, values는 사내.

## 4. Cluster Decision Resource

KubeFed/OCM 통합. 외부 controller가 결정한 cluster 목록 인입.

```yaml
generators:
  - clusterDecisionResource:
      configMapRef: my-decision
      name: production-clusters
      requeueAfterSeconds: 180
```

## 5. PreservedFields

ApplicationSet 재생성 시 child Application의 일부 필드 보존:
```yaml
spec:
  preservedFields:
    annotations: [argocd-image-updater/last-update]
```

Image Updater와 안 부딪힘.

## 6. SyncPolicy in ApplicationSet

```yaml
spec:
  syncPolicy:
    preserveResourcesOnDeletion: false
    applicationsSync: create-update
```

create-update: child Application 삭제 안 함 (수동 cleanup).

## 7. 함정

1. Matrix 폭발.
2. preservedFields 잘못 → 정보 손실.
3. preserveResourcesOnDeletion이 namespace 자원 leak.

## 8. 자가평가

### Q1. Matrix + Plugin?
1. **CMDB + git 결합** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. Multi-source?
1. **chart는 vendor, values는 사내** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. PreservedFields?
1. **재생성 시 일부 field 보존** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. ClusterDecisionResource?
1. **외부 controller가 결정한 cluster 목록** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. 함정?
1. **Matrix 폭발** 2. 안전 3. UI 4. 무관

**정답: 1.**

## 9. 다음

[Week 10: API 통합].
