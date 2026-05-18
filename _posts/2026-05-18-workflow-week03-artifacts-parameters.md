---
layout: post
title: "Workflow Week 3: Artifacts, Parameters, Input/Output 깊게"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series
---

## 학습 목표

- Parameter vs Artifact의 차이를 안다.
- S3/GCS/MinIO에 artifact 저장 설정.
- Artifact passing pattern (input/output, between steps).
- Workflow-level cache (memoization) 활용.

## 1. 비유 — "쪽지 vs 택배"

- Parameter: 작은 메시지(쪽지)를 step 간 전달. 텍스트, 숫자.
- Artifact: 큰 파일/디렉토리(택배)를 전달. 빌드 결과물, 데이터셋.

쪽지는 즉시 전달, 택배는 보관함(S3) 거쳐 전달.

## 2. Artifact backend 설정

`argo-server`의 `artifactRepository` 설정 (configmap `artifact-repositories`):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: artifact-repositories
  namespace: argo
  annotations:
    workflows.argoproj.io/default-artifact-repository: default-v1
data:
  default-v1: |
    s3:
      bucket: mycorp-argo-artifacts
      endpoint: s3.amazonaws.com
      region: us-east-1
      accessKeySecret:
        name: artifact-creds
        key: accessKey
      secretKeySecret:
        name: artifact-creds
        key: secretKey
      keyFormat: "{{workflow.name}}/{{pod.name}}/{{outputs.artifacts.name}}"
```

GCS, Azure Blob, MinIO 모두 지원.

빅테크 표준: IRSA로 정적 키 제거, KMS encryption, lifecycle policy로 90일 후 삭제.

## 3. Output Artifact

```yaml
- name: build
  container:
    image: golang:1.21
    command: [sh, -c]
    args:
      - |
        cd /src && go build -o /tmp/app .
  outputs:
    artifacts:
      - name: binary
        path: /tmp/app
```

Argo가 `/tmp/app`을 S3에 tar.gz로 자동 업로드.

## 4. Input Artifact

```yaml
- name: deploy
  inputs:
    artifacts:
      - name: binary
        path: /tmp/app
  container:
    image: alpine
    command: [sh, -c]
    args: ["./tmp/app deploy"]
```

S3에서 다운로드 후 path에 위치.

## 5. Step 간 연결

```yaml
- name: pipeline
  dag:
    tasks:
      - name: build
        template: build
      - name: deploy
        template: deploy
        dependencies: [build]
        arguments:
          artifacts:
            - name: binary
              from: "{{tasks.build.outputs.artifacts.binary}}"
```

build의 binary가 자동 S3 → deploy의 input.

## 6. Git Artifact

```yaml
inputs:
  artifacts:
    - name: source
      path: /src
      git:
        repo: https://github.com/mycorp/myapp.git
        revision: main
        usernameSecret:
          name: git-creds
          key: username
        passwordSecret:
          name: git-creds
          key: password
```

source code 자동 clone.

## 7. Output에서 directory tar

```yaml
outputs:
  artifacts:
    - name: dist
      path: /workspace/dist
      archive:
        tar:
          compressionLevel: 6
```

전체 디렉토리를 tar.gz로.

## 8. Volume 사용 (artifact 대신)

큰 파일이 같은 노드의 step 간만 공유면 volume이 더 빠름.

```yaml
spec:
  volumes:
    - name: workdir
      emptyDir: {}
  templates:
    - name: build
      container:
        image: ...
        volumeMounts:
          - { name: workdir, mountPath: /workspace }
    - name: test
      container:
        image: ...
        volumeMounts:
          - { name: workdir, mountPath: /workspace }
```

단, 두 step이 다른 노드에 스케줄되면 안 됨. `affinity`로 같은 노드 강제 또는 PVC 사용.

## 9. Memoization (Cache)

같은 입력 → 같은 출력. 두 번째부터 skip.

```yaml
- name: expensive
  memoize:
    key: "{{inputs.parameters.input}}"
    maxAge: "1h"
    cache:
      configMap:
        name: argo-cache
  container: ...
```

빌드 cache, 데이터 처리 결과 cache에 매우 유용.

## 10. Workflow Parameters

```bash
argo submit hello.yaml -p msg=world -p env=prod
```

YAML:
```yaml
spec:
  arguments:
    parameters:
      - { name: msg, value: hello }
      - { name: env, value: dev }
```

`{{workflow.parameters.msg}}` 로 어디서나 참조.

## 11. 변수 종류

- `{{inputs.parameters.X}}`
- `{{inputs.artifacts.X.path}}`
- `{{outputs.parameters.X}}`
- `{{workflow.parameters.X}}`
- `{{workflow.name}}`, `{{workflow.namespace}}`
- `{{steps.X.outputs.parameters.Y}}`
- `{{tasks.X.outputs.parameters.Y}}` (dag)
- `{{item}}` (loop)

## 12. 실습

```bash
# 1. S3/MinIO artifact repo 설정
# 2. build template으로 binary output
# 3. deploy template으로 binary input
# 4. git artifact로 source clone
# 5. memoize로 두 번째 실행 skip 확인
```

## 13. 자가평가 퀴즈

### Q1. Parameter vs Artifact?
1. **Parameter는 작은 텍스트, Artifact는 큰 파일/디렉토리**
2. 같음
3. 반대
4. 무관

**정답: 1.**

### Q2. Output artifact 저장 위치?
1. **S3/GCS/MinIO 등 backend (configmap에 정의)**
2. etcd
3. local
4. 무관

**정답: 1.**

### Q3. Git artifact의 용도?
1. **source code 자동 clone**
2. backup
3. cache
4. 무관

**정답: 1.**

### Q4. Memoize의 가치?
1. **같은 입력의 결과 cache → 두 번째 skip**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. Volume이 artifact보다 빠른 경우?
1. **같은 노드의 step 간 큰 데이터 공유**
2. 다른 노드
3. 무관
4. 의미 없음

**정답: 1.**

## 14. 다음 주차

[Week 4: Workflow 패턴]에서 retry, suspend, exit handler 등 운영 패턴을 다룹니다.
