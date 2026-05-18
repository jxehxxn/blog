---
layout: post
title: "Policy Week 6: Rego 깊게 — Set, Object, Partial Evaluation, Modules"
date: 2026-05-18 20:00:00 +0900
categories: policy opa rego platform senior-series
---

## 학습 목표

- Set/Object/Array 자유 사용.
- Partial vs complete rule.
- Module 구조.
- Rego best practices.

## 1. Set vs Array

```rego
# Array (순서)
arr := [1, 2, 3]

# Set (순서 무관, 중복 없음)
s := {1, 2, 3}

# Set comprehension
unique_images := {c.image | c := input.containers[_]}
```

## 2. Object

```rego
obj := {"a": 1, "b": 2}
val := obj.a  # 1

# Object comprehension
labels := {key: value | some key; value := input.metadata.labels[key]; startswith(key, "app.")}
```

## 3. Partial vs Complete Rule

```rego
# Complete rule (single value)
allowed = false { ... }
allowed = true { ... }

# Partial rule (set of values)
violations[msg] {
  ...
  msg := "..."
}
```

violations는 N개 메시지 set, allowed는 한 값.

## 4. Default

```rego
default allowed = false

allowed {
  # 조건 만족
}
```

조건 미만족 시 default. 누락 방지.

## 5. Helper Function

```rego
is_prod(ns) {
  ns == "prod"
}

# 함수 (else)
get_tier(ns) = "tier1" {
  ns == "prod"
}
get_tier(ns) = "tier2" {
  ns != "prod"
}
```

## 6. Module Import

```rego
package mycorp.k8s.utils

is_prod(ns) { ns == "prod" }
```

```rego
package mycorp.k8s.security

import data.mycorp.k8s.utils

deny[msg] {
  utils.is_prod(input.metadata.namespace)
  not input.spec.securityContext.runAsNonRoot
  msg := "prod requires runAsNonRoot"
}
```

## 7. Test 패턴

```rego
package mycorp.k8s.security_test

import data.mycorp.k8s.security

test_prod_requires_nonRoot {
  count(security.deny) > 0 with input as {
    "metadata": {"namespace": "prod"},
    "spec": {}
  }
}

test_dev_allowed {
  count(security.deny) == 0 with input as {
    "metadata": {"namespace": "dev"},
    "spec": {}
  }
}
```

```bash
opa test policies/
```

## 8. JSON Path / Walk

```rego
deny[msg] {
  walk(input, [path, value])
  path[count(path) - 1] == "image"
  startswith(value, "docker.io/")
  msg := sprintf("forbidden image at path %v", [path])
}
```

`walk`로 임의 깊이 탐색.

## 9. Performance Tip

- Index lookups (`labels[name]`) > linear scan.
- Pre-compute 자주 쓰는 값.
- Partial evaluation (gateway에서).

## 10. 자가평가 퀴즈

### Q1. Set vs Array?
1. **Set 순서 무관, 중복 없음**
2. 같음 3. 반대 4. 무관

**정답: 1.**

### Q2. Default 가치?
1. **조건 미만족 시 누락 방지**
2. 빠름 3. UI 4. 무관

**정답: 1.**

### Q3. Partial rule?
1. **결과가 set/object**
2. single value 3. 함수 4. 무관

**정답: 1.**

### Q4. `walk` 의 용도?
1. **임의 깊이 JSON 탐색**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. test 패턴?
1. **`with input as ...` 로 입력 모킹**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 11. 다음

[Week 7: Mutation 깊게].
