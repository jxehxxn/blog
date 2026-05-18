---
layout: post
title: "Policy 보충 2: Kyverno Mutation 패턴 깊게"
date: 2026-05-18 20:00:00 +0900
categories: policy kyverno mutation platform senior-series supplement
---

7주차 mutation을 패턴 카탈로그로.

## 1. Default Resource Limits

```yaml
mutate:
  patchStrategicMerge:
    spec:
      containers:
        - (name): "*"
          resources:
            +(limits):
              memory: "256Mi"
              cpu: "500m"
            +(requests):
              memory: "128Mi"
              cpu: "100m"
```

LimitRange와 협업.

## 2. Sidecar Injection

```yaml
mutate:
  patchStrategicMerge:
    metadata:
      annotations:
        +(linkerd.io/inject): enabled
    spec:
      +(initContainers):
        - name: setup
          image: mycorp/setup
```

mesh injection 등.

## 3. Image Pull Secret 자동 추가

```yaml
mutate:
  patchStrategicMerge:
    spec:
      +(imagePullSecrets):
        - name: mycorp-registry
```

## 4. Tag → Digest 변환

```yaml
verifyImages:
  - imageReferences: ["mycorp/*"]
    mutateDigest: true
```

빌드 시 tag로 deploy → admission이 digest로 변환. drift 방지.

## 5. Service ↔ Pod Label 동기화

```yaml
mutate:
  foreach:
    - list: "request.object.spec.containers"
      patchesJson6902: |
        - op: add
          path: /metadata/labels/app
          value: "{{request.object.metadata.labels.app}}"
```

## 6. Namespace 생성 시 Quota 자동

```yaml
generate:
  kind: ResourceQuota
  name: default-quota
  namespace: "{{request.object.metadata.name}}"
  data:
    spec:
      hard:
        requests.cpu: "100"
        requests.memory: "200Gi"
```

(generate. 사실 mutation은 아니지만 패턴 카탈로그.)

## 7. nodeSelector 강제

```yaml
mutate:
  patchStrategicMerge:
    spec:
      nodeSelector:
        workload: "{{request.namespace}}"
```

namespace별 노드 분리.

## 8. 사내 표준 라벨

```yaml
mutate:
  patchStrategicMerge:
    metadata:
      labels:
        +(team): "{{request.namespace}}"
        +(cost-center): "auto-detected"
```

## 9. 운영 함정

1. mutate 후 validate fail → 무한 reconcile (controller 측).
2. patch 충돌 (여러 mutation policy).
3. existing 자원 mutation 비용.

## 10. 결론

Mutation은 default + 강제의 균형. 운영 카탈로그를 잘 관리하면 보일러플레이트 코드 0.
