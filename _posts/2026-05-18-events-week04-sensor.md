---
layout: post
title: "Events Week 4: Sensor + Trigger"
date: 2026-05-18 23:00:00 +0900
categories: argo-events platform senior-series
---

## 1. Sensor CR

```yaml
spec:
  dependencies:
    - name: push-dep
      eventSourceName: github
      eventName: mycorp-myapp
  triggers:
    - template:
        name: workflow-trigger
        k8s:
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata: { generateName: build- }
              spec: ...
```

dependency = 어떤 event 기다림. trigger = 무엇 할지.

## 2. Trigger 종류
- `k8s`: K8s 자원 create/update/patch.
- `argoWorkflow`: Workflow submit.
- `http`: HTTP webhook.
- `aws.lambda`.
- `kafka`: 메시지 produce.
- `slack`.
- `custom`.

## 3. Multi-dep

```yaml
dependencies:
  - name: a
    ...
  - name: b
    ...
triggers:
  - template:
      conditions: "a && b"  # 둘 다 와야
      ...
```

dependency logic.

## 4. Parameters from Event

```yaml
triggers:
  - template:
      argoWorkflow:
        operation: submit
        source: ...
        parameters:
          - src:
              dependencyName: push-dep
              dataKey: body.head_commit.id
            dest: spec.arguments.parameters.0.value
```

JSONPath로 event body에서 추출.

## 5. 자가평가
### Q1. dependency 의미? **기다리는 event**. 정답 1.
### Q2. trigger 종류? **k8s/workflow/http/lambda/kafka 등**. 정답 1.
### Q3. multi-dep AND? **conditions: "a && b"**. 정답 1.
### Q4. parameter src? **dependency + dataKey**. 정답 1.
### Q5. operation submit? **Workflow trigger**. 정답 1.

## 다음
[Week 5: GitHub/GitLab source].
