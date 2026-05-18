---
layout: post
title: "Policy 보충 1: Rego 심화 — Module, Partial Evaluation, Performance"
date: 2026-05-18 20:00:00 +0900
categories: policy opa rego platform senior-series supplement
---

6주차 Rego를 심화로.

## 1. Module Organization

```
policies/
  lib/
    utils.rego           # 공용
    images.rego
  k8s/
    pods.rego
    services.rego
```

`import data.lib.utils` 로 공유.

## 2. Partial Evaluation

조건 일부만 평가해 최적화.

```bash
opa eval --partial 'data.k8s.allow' --unknowns input.kind
# 'kind != X' 같은 부분 결과
```

OPA gateway가 활용해 latency 줄임.

## 3. Performance

- `every`: 모든 요소 만족 검사 (efficient).
- Indexed access (`obj[key]`) > linear (`obj[_].key == "x"`).
- Pre-built indexed input.

## 4. Built-in Functions

- `json.patch`, `json.diff`
- `regex.match`, `regex.find_all`
- `http.send` (제한적, 신중)
- `time.now_ns`
- `crypto.sha256`

## 5. Bundle 배포

OPA bundle (tar.gz) 로 정책 + 데이터 묶음. CDN에서 배포.

```bash
opa run --server -b bundle.tar.gz
```

원격 bundle pull도 가능 (HTTP).

## 6. Decision Logging

policy 평가 결과를 외부 시스템에 로그.

```bash
opa run -c config.yaml
```

```yaml
decision_logs:
  service: log-service
```

audit + ML 분석.

## 7. 결론

Rego는 단순 표현 → 복잡 도메인 로직까지 다 됨. 시니어는 module + partial eval + performance + bundle 운영.
