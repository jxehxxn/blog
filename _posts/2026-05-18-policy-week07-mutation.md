---
layout: post
title: "Policy Week 7: Mutation 깊게 — Strategic Merge, JSON Patch, ForEach"
date: 2026-05-18 20:00:00 +0900
categories: policy kyverno mutation platform senior-series
---

## 학습 목표

- Mutation의 3가지 방식.
- Sidecar injection 패턴.
- Default 값 주입.
- Conditional mutation.

## 1. 비유 — "자동 보정 펜"

PR 들어오면 자동으로 빠진 필드 보충, 잘못된 형식 수정. Reviewer 부담 줄임.

## 2. Strategic Merge Patch

K8s 표준 patch. 같은 key는 overlay.

```yaml
mutate:
  patchStrategicMerge:
    metadata:
      labels:
        +(managed-by): kyverno
    spec:
      containers:
        - (name): "*"
          imagePullPolicy: IfNotPresent
```

`+(name)`: 없으면 추가. `(name): "*"`: 모든 container.

## 3. JSON Patch

```yaml
mutate:
  patchesJson6902: |
    - op: add
      path: /metadata/labels/env
      value: prod
    - op: replace
      path: /spec/replicas
      value: 3
```

명시적 path. 강력하지만 verbose.

## 4. ForEach

list에 반복.

```yaml
mutate:
  foreach:
    - list: request.object.spec.containers
      patchStrategicMerge:
        spec:
          containers:
            - name: "{{ element.name }}"
              resources:
                limits:
                  memory: "256Mi"
                  cpu: "500m"
```

## 5. 자주 쓰는 mutation

### Default labels
```yaml
patchStrategicMerge:
  metadata:
    labels:
      +(team): unknown
      +(env): "{{ request.namespace }}"
```

### Sidecar injection
```yaml
patchStrategicMerge:
  spec:
    +(initContainers):
      - name: setup
        image: mycorp/setup
        ...
    containers:
      - (name): "*"
        env:
          - name: ENV
            value: "prod"
```

### NodeSelector 강제
```yaml
patchStrategicMerge:
  spec:
    +(nodeSelector):
      workload: "{{ request.object.metadata.labels.tier }}"
```

## 6. Conditional Mutation

```yaml
mutate:
  patchStrategicMerge:
    spec:
      containers:
        - (name): "*"
          (image): "mycorp/*"   # mycorp/* image만
          imagePullPolicy: Always
```

## 7. Mutate Existing

신규뿐 아니라 기존 리소스도 변경.

```yaml
mutateExistingOnPolicyUpdate: true
mutate:
  targets:
    - apiVersion: v1
      kind: Pod
      namespace: payments
  patchStrategicMerge: ...
```

policy 업데이트 시 기존 자원도 update.

## 8. 운영 함정

1. mutate strategic merge가 다른 controller와 충돌.
2. JSON patch path 오류.
3. mutate 결과 validation에 fail (mutate 후 validate).
4. existing mutation 대량 (cluster 부담).
5. mutation chain 순서 모호.

## 9. Best Practice

- Mutation은 default 값 추가에 우선 사용.
- 강제 변경은 validation으로.
- ChangeLog dashboard로 mutation 추적.

## 10. 자가평가 퀴즈

### Q1. Strategic Merge Patch 의미?
1. **같은 key overlay, list element는 strategic key 기반 merge**
2. JSON 3. UI 4. 무관

**정답: 1.**

### Q2. `+(name)`?
1. **없으면 추가**
2. 무시 3. UI 4. 무관

**정답: 1.**

### Q3. ForEach 가치?
1. **list 반복 mutation**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. mutateExistingOnPolicyUpdate?
1. **policy 업데이트 시 기존 자원도 변경**
2. 무관 3. UI 4. 빠름

**정답: 1.**

### Q5. Mutation 우선 용도?
1. **default 값 추가**
2. 강제 거부 3. UI 4. 무관

**정답: 1.**

## 11. 다음

[Week 8: Image admission + 공급망].
