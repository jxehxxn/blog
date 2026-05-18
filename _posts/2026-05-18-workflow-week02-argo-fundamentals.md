---
layout: post
title: "Workflow Week 2: Argo Workflows 기초 — Workflow CR, Templates, Steps/DAG"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series
---

## 학습 목표

- Argo Workflows 설치 + 첫 workflow 실행.
- Workflow CR의 구조 한 줄도 빠짐없이 본다.
- Template 종류(container, script, steps, dag, resource, suspend)를 안다.
- Steps vs DAG 표현의 차이.

## 1. 설치

```bash
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.0/install.yaml

# CLI
brew install argo
argo version
```

UI:
```bash
kubectl -n argo port-forward svc/argo-server 2746:2746
# https://localhost:2746
```

## 2. 첫 Workflow

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-
spec:
  entrypoint: main
  templates:
    - name: main
      container:
        image: alpine:3.18
        command: [echo]
        args: ["hello from workflow"]
```

```bash
argo submit -n argo hello.yaml --watch
```

UI에서 step 진행 시각화.

## 3. Workflow CR 한 줄씩

```yaml
spec:
  entrypoint: main             # 시작 template 이름
  serviceAccountName: argo-wf  # Pod이 쓸 SA
  ttlStrategy:
    secondsAfterCompletion: 3600  # 완료 후 1시간 뒤 자동 삭제
  podGC:
    strategy: OnPodCompletion  # Pod도 자동 정리
  arguments:
    parameters:
      - name: msg
        value: hello
  templates:
    - name: main
      inputs:
        parameters:
          - name: msg
      ...
```

## 4. Template 6종

### container
표준. Pod 1개 실행.
```yaml
- name: hello
  container:
    image: alpine
    command: [echo]
    args: [hello]
```

### script
sh/python 인라인.
```yaml
- name: py
  script:
    image: python:3.11
    command: [python]
    source: |
      print("hello from python")
```

### steps
순차/병렬 (간단).
```yaml
- name: pipeline
  steps:
    - - { name: a, template: stepA }
      - { name: b, template: stepB }      # 같은 -- 안: 병렬
    - - { name: c, template: stepC }      # 위 둘 끝난 후 c
```

### dag
의존성 그래프 (정밀).
```yaml
- name: pipeline
  dag:
    tasks:
      - { name: a, template: stepA }
      - { name: b, template: stepB }
      - { name: c, template: stepC, dependencies: [a, b] }
```

### resource
K8s resource를 만들거나 patch.
```yaml
- name: create-cm
  resource:
    action: create
    manifest: |
      apiVersion: v1
      kind: ConfigMap
      metadata: { name: foo }
      data: { key: value }
```

### suspend
사람 승인 대기.
```yaml
- name: approve
  suspend:
    duration: "1h"   # 또는 무제한
```

## 5. Steps vs DAG

| 항목 | steps | dag |
|------|-------|-----|
| 표현 | 순차/평행 그룹 | 의존성 그래프 |
| 단순성 | 쉬움 | 중간 |
| 복잡 의존성 | 어려움 | 자연스러움 |
| 빅테크 선호 | 단순 case | 복잡 case |

대부분 DAG가 권장. 의존성을 명시적으로 표현.

## 6. 입출력 — Parameters

```yaml
templates:
  - name: main
    steps:
      - - name: hi
          template: greet
          arguments:
            parameters:
              - name: name
                value: "world"
  - name: greet
    inputs:
      parameters:
        - name: name
    container:
      image: alpine
      command: [sh, -c]
      args: ["echo hello {{inputs.parameters.name}}"]
```

`{{inputs.parameters.name}}` 같은 변수 치환이 핵심.

## 7. Output 캡처

```yaml
- name: produce
  container:
    image: alpine
    command: [sh, -c, "echo my-result > /tmp/out.txt"]
  outputs:
    parameters:
      - name: result
        valueFrom: { path: /tmp/out.txt }

- name: consume
  inputs:
    parameters:
      - name: input
  ...
```

DAG로 연결:
```yaml
dag:
  tasks:
    - name: p, template: produce
    - name: c, template: consume
      dependencies: [p]
      arguments:
        parameters:
          - name: input
            value: "{{tasks.p.outputs.parameters.result}}"
```

## 8. Argo CLI

```bash
argo list                  # 최근 workflow
argo submit hello.yaml
argo get <wf>
argo logs <wf>
argo logs <wf> -f          # tail
argo terminate <wf>
argo retry <wf>             # 실패한 step부터 재시도
argo resubmit <wf>          # 처음부터 다시
```

## 9. 빅테크 자주 쓰는 옵션

- `parallelism`: 동시 실행 step 한도.
- `activeDeadlineSeconds`: 전체 timeout.
- `podMetadata.labels`: 모든 Pod 라벨.
- `podPriorityClassName`: priorityClass.
- `nodeSelector`/`affinity`/`tolerations`: 특정 노드 풀.
- `imagePullSecrets`: private registry.

## 10. 실습

```bash
# 1. hello workflow 실행
# 2. steps로 a → (b∥c) → d 표현
# 3. 동일을 dag로 표현
# 4. parameter 입출력
# 5. suspend로 사람 승인 시뮬레이션
```

## 11. 자가평가 퀴즈

### Q1. Workflow의 entrypoint?
1. **시작 template 이름**
2. SA
3. 이미지
4. 무관

**정답: 1.**

### Q2. DAG가 steps보다 권장되는 상황?
1. **복잡한 의존성 표현**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q3. Output 캡처 방법?
1. **outputs.parameters.valueFrom.path (파일에서 읽기)**
2. stdin
3. env
4. 무관

**정답: 1.**

### Q4. argo retry vs resubmit?
1. **retry: 실패한 step부터, resubmit: 처음부터**
2. 같음
3. 반대
4. 무관

**정답: 1.**

### Q5. suspend template의 용도?
1. **사람 승인/외부 신호 대기**
2. 디버깅
3. UI
4. 무관

**정답: 1.**

## 12. 다음 주차

[Week 3: Artifacts]에서 step 간 큰 파일 전달, S3/GCS 통합, cache 패턴을 다룹니다.
