---
layout: post
title: "Policy Week 4: Kyverno 기초 — YAML로 작성하는 정책"
date: 2026-05-18 20:00:00 +0900
categories: policy kyverno platform senior-series
---

## 학습 목표

- Kyverno 설치 + 첫 정책.
- 4 동작 (validate, mutate, generate, verifyImages).
- 매칭 + 예외.

## 1. 비유 — "YAML 법전"

Rego 학습 부담 없이 YAML만으로 정책. K8s 친화적.

## 2. 설치

```bash
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

## 3. Validate Policy

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: require-labels }
spec:
  validationFailureAction: enforce  # 또는 audit
  background: true                  # 기존도 검사
  rules:
    - name: check-owner-label
      match:
        any:
          - resources: { kinds: [Namespace] }
      validate:
        message: "namespace requires owner label"
        pattern:
          metadata:
            labels:
              owner: "?*"
```

`?*`: 임의 문자열 (anything).

## 4. Mutate Policy

요청을 수정.

```yaml
spec:
  rules:
    - name: add-default-resource
      match:
        any: [{ resources: { kinds: [Pod] } }]
      mutate:
        patchStrategicMerge:
          spec:
            containers:
              - (name): "*"
                resources:
                  limits:
                    memory: "256Mi"
                    cpu: "500m"
```

Pod에 resource limit이 없으면 자동 추가.

## 5. Generate Policy

리소스 자동 생성.

```yaml
spec:
  rules:
    - name: default-deny-netpol
      match:
        any: [{ resources: { kinds: [Namespace] } }]
      generate:
        kind: NetworkPolicy
        name: default-deny
        namespace: "{{request.object.metadata.name}}"
        data:
          spec:
            podSelector: {}
            policyTypes: [Ingress, Egress]
```

신규 namespace 생성 시 default-deny NetworkPolicy 자동.

## 6. VerifyImages

```yaml
spec:
  rules:
    - name: verify-cosign
      match: { any: [{ resources: { kinds: [Pod] } }] }
      verifyImages:
        - imageReferences: ["mycorp/*"]
          attestors:
            - entries:
                - keyless:
                    issuer: https://accounts.google.com
                    subject: ci-bot@mycorp.iam.gserviceaccount.com
```

cosign signature 검증. Tier 1 Course 7에서 깊게.

## 7. Background Scan

`background: true` → 기존 리소스도 정기 검사. PolicyReport CR로 결과.

## 8. 패턴 매칭 문법

```yaml
pattern:
  spec:
    replicas: "<=10"             # 비교
    containers:
      - image: "!docker.io/*"    # 거부
        ports:
          - containerPort: 80
        (name): "main"           # 조건 매칭
```

JMESPath + Kyverno 확장.

## 9. context (외부 데이터)

```yaml
spec:
  rules:
    - name: ...
      context:
        - name: nsLabels
          apiCall:
            urlPath: "/api/v1/namespaces/{{request.namespace}}"
            jmesPath: "metadata.labels"
      validate: ...
```

다른 자원 조회.

## 10. 운영 함정

1. `enforce` 처음부터 → 기존 위반 차단.
2. `background: false` → 기존 위반 모름.
3. mutate strategic merge 충돌.
4. generate가 다른 controller와 race.
5. context cache 미설정 → API 부담.

## 11. 자가평가 퀴즈

### Q1. Kyverno 강점?
1. **YAML만으로 정책 작성**
2. Rego 3. UI 4. 무관

**정답: 1.**

### Q2. Mutate 가치?
1. **요청 자체 수정 (default 추가 등)**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q3. Generate 가치?
1. **신규 리소스 자동 생성 (default NetworkPolicy 등)**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q4. VerifyImages?
1. **cosign signature 검증**
2. UI 3. 무관 4. SBOM

**정답: 1.**

### Q5. background scan 가치?
1. **기존 리소스 위반 발견**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 12. 다음

[Week 5: OPA vs Kyverno 비교].
