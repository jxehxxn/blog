---
layout: post
title: "Workflow Week 6: Tekton 기초 — Task, Pipeline, PipelineRun"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series
---

## 학습 목표

- Tekton 설치 + 첫 Pipeline 실행.
- Task / Pipeline / PipelineRun 3 layer 구조.
- Workspaces로 step 간 디렉토리 공유.
- 결과(result) 전달.

## 1. 설치

```bash
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# CLI
brew install tektoncd-cli
tkn version
```

Dashboard:
```bash
kubectl apply -f https://github.com/tektoncd/dashboard/releases/latest/download/release.yaml
```

## 2. 3 Layer

```
[Task]          # 재사용 가능한 step 단위 (예: git-clone, build, test)
   ↓
[Pipeline]      # Task의 조합 (DAG)
   ↓
[PipelineRun]   # Pipeline의 한 번 실행 (인스턴스)
```

Task는 "함수", Pipeline은 "프로그램", PipelineRun은 "실행 결과".

## 3. Task

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata: { name: hello }
spec:
  steps:
    - name: echo
      image: alpine:3.18
      script: |
        echo hello tekton
```

TaskRun으로 단독 실행 가능:
```bash
tkn task start hello
```

## 4. Pipeline

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata: { name: build-and-test }
spec:
  params:
    - name: repo-url
    - name: revision
      default: main
  workspaces:
    - name: source
  tasks:
    - name: fetch
      taskRef: { name: git-clone }
      workspaces:
        - { name: output, workspace: source }
      params:
        - { name: url, value: "$(params.repo-url)" }
        - { name: revision, value: "$(params.revision)" }
    - name: build
      taskRef: { name: golang-build }
      runAfter: [fetch]
      workspaces:
        - { name: source, workspace: source }
    - name: test
      taskRef: { name: golang-test }
      runAfter: [build]
      workspaces:
        - { name: source, workspace: source }
```

## 5. PipelineRun

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata: { generateName: build-and-test- }
spec:
  pipelineRef: { name: build-and-test }
  params:
    - { name: repo-url, value: "https://github.com/mycorp/myapp.git" }
  workspaces:
    - name: source
      volumeClaimTemplate:
        spec:
          accessModes: [ReadWriteOnce]
          resources: { requests: { storage: 1Gi } }
```

```bash
kubectl create -f pr.yaml
tkn pr logs -f
```

## 6. Workspaces

step 간 디렉토리 공유. PVC 또는 emptyDir 또는 ConfigMap/Secret.

```yaml
spec:
  workspaces:
    - name: source
      description: "Code workspace"
      mountPath: /workspace/source
```

PV로 backing: 같은 노드 강제 (RWO) 또는 RWX storage class.

## 7. Results — 작은 출력값

```yaml
spec:
  results:
    - name: image-digest
      description: built image digest
  steps:
    - script: |
        ...
        echo -n "sha256:abc..." > $(results.image-digest.path)
```

다른 Task에서:
```yaml
params:
  - name: digest
    value: "$(tasks.build.results.image-digest)"
```

## 8. Catalog — Tekton Hub

`hub.tekton.dev` 에 표준 Task 수백 개. git-clone, kaniko, helm-deploy, golangci-lint 등.

```bash
tkn hub install task git-clone
```

빅테크 표준: 사내 fork + 자체 catalog.

## 9. 일반 패턴 — CI Pipeline

```yaml
tasks:
  - fetch (git-clone)
  - lint (golangci-lint)
  - test (go-test)
  - build (kaniko build + push)
  - scan (trivy)
  - sign (cosign)
  - deploy (kubectl apply 또는 ArgoCD trigger)
```

각 단계가 Task. catalog 활용으로 boilerplate 최소.

## 10. Failure / Retries

```yaml
tasks:
  - name: flaky
    taskRef: { name: external-api }
    retries: 3
```

전체 Pipeline timeout:
```yaml
spec:
  timeouts:
    pipeline: "1h"
    tasks: "30m"
    finally: "10m"
```

## 11. Finally — 항상 실행

Argo의 exit handler 대응:
```yaml
spec:
  tasks: [...]
  finally:
    - name: notify
      taskRef: { name: slack-message }
      params:
        - { name: message, value: "Pipeline $(context.pipelineRun.name) completed: $(tasks.status)" }
```

## 12. 실습

```bash
# 1. Tekton 설치
# 2. hello Task
# 3. git-clone catalog 설치 후 fetch
# 4. build + test Pipeline
# 5. PipelineRun으로 실행
# 6. Results로 image digest 전달
```

## 13. 자가평가 퀴즈

### Q1. Tekton 3 layer?
1. **Task / Pipeline / PipelineRun**
2. Step / Workflow / Run
3. UI
4. 무관

**정답: 1.**

### Q2. Workspace의 역할?
1. **step 간 디렉토리 공유**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q3. Results 의 용도?
1. **작은 출력값을 다음 Task에 전달**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q4. Tekton Hub의 가치?
1. **표준 Task 카탈로그 → 재사용**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. Finally 의 역할?
1. **Argo exit handler 대응 — 항상 실행 cleanup**
2. 빠름
3. UI
4. 무관

**정답: 1.**

## 14. 다음 주차

[Week 7: Tekton Triggers + Resolvers + Chains]에서 외부 이벤트 trigger와 supply chain 보안을 다룹니다.
