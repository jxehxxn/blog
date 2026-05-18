---
layout: post
title: "Workflow 보충 1: 패턴 깊게 — DAG, Fan-out/in, MapReduce, Event-driven"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series supplement
---

4주차 패턴을 ML/data 영역까지 확장합니다.

## 1. MapReduce 패턴

대량 데이터 처리:
1. **Map**: input을 N개 chunk로 split.
2. **Process**: N개 worker가 병렬 처리.
3. **Reduce**: 결과 N개 → 1개로 merge.

```yaml
dag:
  tasks:
    - name: split
      template: split    # outputs: list of chunk paths
    - name: process
      template: process
      dependencies: [split]
      withParam: "{{tasks.split.outputs.parameters.chunks}}"
      arguments:
        parameters:
          - { name: chunk, value: "{{item}}" }
    - name: reduce
      template: reduce
      dependencies: [process]
      arguments:
        artifacts:
          - { name: all, from: "{{tasks.process.outputs.artifacts.result}}" }
```

ML 데이터셋 전처리, 로그 집계 등.

## 2. Diamond DAG

```
        A
       / \
      B   C
       \ /
        D
```

B와 C가 병렬, D는 둘 다 끝나야:
```yaml
tasks:
  - { name: a, template: A }
  - { name: b, template: B, dependencies: [a] }
  - { name: c, template: C, dependencies: [a] }
  - { name: d, template: D, dependencies: [b, c] }
```

## 3. 조건부 분기

```yaml
- name: deploy
  template: deploy
  when: "{{tasks.test.outputs.parameters.result}} == success"
  dependencies: [test]
```

조건 거짓이면 task skip.

## 4. Conditional Subworkflow

큰 분기는 sub-workflow:
```yaml
- name: prod-pipeline
  template: prod-flow
  when: "{{workflow.parameters.env}} == prod"
```

## 5. Recursion

workflow가 자기 자신을 step으로 호출 → 재귀.

```yaml
templates:
  - name: recurse
    inputs:
      parameters: [{name: n}]
    steps:
      - - name: print
          template: print
          arguments: { parameters: [{name: msg, value: "{{inputs.parameters.n}}"}] }
      - - name: next
          template: recurse
          when: "{{inputs.parameters.n}} > 0"
          arguments:
            parameters:
              - { name: n, value: "{{= inputs.parameters.n - 1}}" }
```

ML hyperparameter search 등.

## 6. Event-driven Loop

Argo Events로 message 받음 → workflow 생성. workflow는 한 message 처리. 메시지마다 새 workflow → 자연스러운 scale.

## 7. Long-running ML Training

```yaml
- name: train
  container:
    image: pytorch/pytorch:2.0
    resources:
      limits: { nvidia.com/gpu: 4 }
    command: ["python", "train.py"]
  retryStrategy:
    limit: 3
    retryPolicy: OnFailure
  activeDeadlineSeconds: 86400   # 24시간
```

GPU resource + 긴 timeout.

## 8. Workflow-of-Workflows

Master workflow가 여러 sub workflow를 submit:
```yaml
- name: master
  steps:
    - - name: sub
        template: submit-sub
- name: submit-sub
  resource:
    action: create
    manifest: |
      apiVersion: argoproj.io/v1alpha1
      kind: Workflow
      ...
```

## 9. Cron + Event 결합

CronWorkflow가 매일 + 추가로 webhook으로 즉시 trigger 가능.

## 10. 결론

DAG · loop · 조건 · recursion · event 결합으로 거의 모든 워크플로 표현 가능. 빅테크 ML/data 파이프라인이 Argo Workflows 주력 영역인 이유.
