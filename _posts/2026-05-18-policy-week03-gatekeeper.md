---
layout: post
title: "Policy Week 3: Gatekeeper 아키텍처 + ConstraintTemplate"
date: 2026-05-18 20:00:00 +0900
categories: policy opa gatekeeper platform senior-series
---

## 학습 목표

- Gatekeeper 설치 + admission webhook 흐름.
- ConstraintTemplate vs Constraint 분리.
- Audit + Sync로 기존 위반 발견.

## 1. 설치

```bash
helm install gatekeeper gatekeeper/gatekeeper \
  -n gatekeeper-system --create-namespace
```

webhook 자동 등록.

## 2. ConstraintTemplate

policy "코드" 정의.

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata: { name: k8srequiredlabels }
spec:
  crd:
    spec:
      names: { kind: K8sRequiredLabels }
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items: { type: string }
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          required := input.parameters.labels
          provided := input.review.object.metadata.labels
          missing := required[_]
          not provided[missing]
          msg := sprintf("missing label %v", [missing])
        }
```

## 3. Constraint

policy "인스턴스".

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata: { name: ns-must-have-owner }
spec:
  match:
    kinds:
      - { apiGroups: [""], kinds: [Namespace] }
  parameters:
    labels: [owner, env]
```

Namespace에 `owner`, `env` 라벨 의무.

## 4. 매칭

```yaml
match:
  kinds: [...]
  namespaces: [team-payments]
  excludedNamespaces: [kube-system]
  scope: Namespaced
  labelSelector: ...
```

## 5. Enforcement Mode

```yaml
spec:
  enforcementAction: deny    # 또는 warn, dryrun
```

- `deny`: 거부.
- `warn`: 통과시키되 경고.
- `dryrun`: audit만.

빅테크 패턴: 신규 정책은 `dryrun` → `warn` → `deny` 점진.

## 6. Audit

`gatekeeper-audit` Pod이 주기적으로 기존 리소스 검사 → ConstraintStatus 업데이트.

```bash
kubectl get constraint ns-must-have-owner -o yaml | grep totalViolations
```

기존 위반 발견. baseline 정리.

## 7. Sync (mutation/external)

Gatekeeper는 다른 cluster 리소스 참조 가능 (예: 다른 namespace의 ConfigMap):

```yaml
apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata: { name: config, namespace: gatekeeper-system }
spec:
  sync:
    syncOnly:
      - { group: "", version: v1, kind: ConfigMap }
```

이후 Rego에서 `data.inventory.cluster.<kind>` 로 접근.

## 8. Gatekeeper Library

`open-policy-agent/gatekeeper-library`: 표준 ConstraintTemplate 30+.

빅테크 표준: library 가져와 + 사내 커스텀.

## 9. 운영 함정

1. webhook 자체 장애 시 API server fail.
2. `enforcementAction: deny`를 처음부터 → 기존 위반 다 deny.
3. audit 빈도 부담.
4. sync 너무 많은 자원 → 메모리.
5. ConstraintTemplate breaking change.

## 10. 자가평가 퀴즈

### Q1. ConstraintTemplate vs Constraint?
1. **Template = 코드, Constraint = 인스턴스**
2. 같음
3. 반대
4. 무관

**정답: 1.**

### Q2. enforcementAction 종류?
1. **deny / warn / dryrun**
2. UI
3. 무관
4. allow only

**정답: 1.**

### Q3. Audit의 가치?
1. **기존 리소스 위반 발견 → baseline 정리**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q4. Sync 가치?
1. **다른 자원 참조 (Rego에서 inventory 접근)**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. 빅테크 점진 도입?
1. **dryrun → warn → deny**
2. 한 번에 deny
3. UI
4. 무관

**정답: 1.**

## 11. 다음

[Week 4: Kyverno 기초].
