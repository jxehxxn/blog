---
layout: post
title: "Webhook Week 6: Namespace / Object Selector"
date: 2026-05-19 05:00:00 +0900
categories: admission webhook senior-series
---

## namespaceSelector
특정 namespace만 적용:
```yaml
namespaceSelector:
  matchLabels: { policy-enforce: "true" }
```

## objectSelector
객체 라벨 기반:
```yaml
objectSelector:
  matchLabels: { tier: prod }
```

## rules
어떤 verb/resource:
```yaml
rules:
  - operations: [CREATE, UPDATE]
    apiGroups: [""]
    resources: [pods]
```

## 자가평가
### Q1. namespaceSelector? **namespace 기반 매칭**. 정답 1.
### Q2. objectSelector? **객체 라벨 기반**. 정답 1.
### Q3. rules? **verb/resource 매칭**. 정답 1.
### Q4. exclude? **selector 역으로**. 정답 1.
### Q5. kube-system exclude? **표준 (admission 자체 보호)**. 정답 1.
