---
layout: post
title: "Events Week 9: Filter + Parameter 깊게"
date: 2026-05-18 23:00:00 +0900
categories: argo-events platform senior-series
---

## Filter 종류

```yaml
dependencies:
  - name: dep
    filters:
      data:
        - path: body.ref
          type: string
          value: ["refs/heads/main"]
      ctx:
        - path: type
          type: string
          value: ["push"]
      time:
        - { start: "09:00:00", stop: "18:00:00" }   # 업무 시간만
```

## Logic

```yaml
filtersLogicalOperator: and  # 또는 or
```

## Parameters

event → trigger 변환:

```yaml
triggers:
  - template:
      argoWorkflow:
        ...
      parameters:
        - src:
            dependencyName: push
            dataKey: body.head_commit.id
          dest: spec.arguments.parameters.0.value
```

JSONPath로 event body에서 추출.

## Conditional Trigger

```yaml
triggers:
  - template:
      ...
    conditions: "dep_a && !dep_b"
```

dependency 조합 logic.

## 자가평가
### Q1. data filter? **body.* JSONPath**. 정답 1.
### Q2. time filter? **업무 시간 등**. 정답 1.
### Q3. parameters? **event → trigger value**. 정답 1.
### Q4. conditions? **dep 조합 logic**. 정답 1.
### Q5. logical op? **and/or**. 정답 1.

## 다음
[Week 10: 보안].
