---
layout: post
title: "Policy Week 2: OPA 기초 + Rego 첫걸음"
date: 2026-05-18 20:00:00 +0900
categories: policy opa rego platform senior-series
---

## 학습 목표

- OPA 단독 사용.
- Rego 기본 (rule, default, comprehension).
- conftest로 plan/manifest 검사.

## 1. OPA 단독

```bash
brew install opa

cat > policy.rego <<EOF
package myapp

default allow = false

allow {
  input.method == "GET"
  input.path[0] == "public"
}
EOF

cat > input.json <<EOF
{"method": "GET", "path": ["public", "data.json"]}
EOF

opa eval --data policy.rego --input input.json "data.myapp.allow"
# true
```

## 2. Rego 기본

```rego
package mycorp.k8s

# 기본 deny
deny[msg] {
  input.kind == "Pod"
  not input.spec.securityContext.runAsNonRoot
  msg := sprintf("Pod %v must runAsNonRoot", [input.metadata.name])
}

# Helper rule
container_has_limit(c) {
  c.resources.limits.memory
  c.resources.limits.cpu
}

deny[msg] {
  input.kind == "Pod"
  c := input.spec.containers[_]
  not container_has_limit(c)
  msg := sprintf("Container %v missing resource limit", [c.name])
}
```

핵심:
- `package`: 네임스페이스.
- `deny[msg]`: deny 가능한 메시지 집합.
- `_`: 임의 인덱스.
- `not`: 조건 부정.
- `:=` 할당, `=` 같음.

## 3. Conftest

Terraform plan, K8s manifest, JSON 어떤 파일이든 검사.

```bash
conftest test deployment.yaml --policy policies/
```

CI에 통합. 5주차 (IaC) 와 동일 패턴.

## 4. Comprehension

```rego
images := [c.image | c := input.spec.containers[_]]
forbidden := [img | img := images[_]; startswith(img, "docker.io/")]
```

list/set/object comprehension.

## 5. Iteration

```rego
deny[msg] {
  some i
  c := input.spec.containers[i]
  not c.resources.limits.memory
  msg := sprintf("container[%d] missing limit", [i])
}
```

`some`으로 변수 선언.

## 6. Functions

```rego
is_prod(ns) {
  ns == "prod" 
} else = false
```

또는:
```rego
get_image_repo(image) = repo {
  parts := split(image, "/")
  repo := parts[0]
}
```

## 7. Testing

```rego
package mycorp.k8s_test

import data.mycorp.k8s

test_deny_no_runAsNonRoot {
  count(k8s.deny) > 0 with input as {"kind": "Pod", "metadata": {"name": "p"}, "spec": {}}
}
```

`opa test policies/`로 unit test.

## 8. 함정

1. `=` (같음) vs `:=` (할당) 혼동.
2. `not` 의 정확한 의미 (rule 미평가시 not은 true).
3. iteration `_` vs `some`.
4. set vs list.
5. partial vs complete rule.

## 9. 자가평가 퀴즈

### Q1. `deny[msg]` 의 의미?
1. **msg가 들어 있는 deny set 만들기**
2. 함수
3. UI
4. 무관

**정답: 1.**

### Q2. `_` 의 의미?
1. **임의 인덱스 (iteration)**
2. 무시
3. UI
4. 무관

**정답: 1.**

### Q3. `:=` vs `=`?
1. **`:=` 할당, `=` 같음 비교**
2. 같음
3. UI
4. 무관

**정답: 1.**

### Q4. Conftest 가치?
1. **Rego policy를 임의 파일에 적용 (TF plan, manifest)**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q5. `opa test`?
1. **policy unit test**
2. UI
3. 빠름
4. 무관

**정답: 1.**

## 10. 다음

[Week 3: Gatekeeper 아키텍처].
