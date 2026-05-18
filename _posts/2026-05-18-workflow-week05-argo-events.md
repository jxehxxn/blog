---
layout: post
title: "Workflow Week 5: Argo Events — 외부 이벤트 → Workflow Trigger"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series argo-events
---

## 학습 목표

- Argo Events 3컴포넌트(EventBus, EventSource, Sensor) 책임 분리를 안다.
- 웹훅/S3/Kafka 등 다양한 source로 trigger 구성.
- Sensor 조건으로 workflow 생성.
- GitOps 패턴과 결합.

## 1. 비유 — "공장 알림 시스템"

비유: 공장에 (1) 알림 기기(EventSource - 사이렌, 전화, 이메일), (2) 알림 버스(EventBus - 무전기), (3) 반응 로봇(Sensor - "이런 알림이면 이 작업 시작") 이 있는 구조.

## 2. 3 컴포넌트

### EventBus
NATS streaming 기반 메시지 버스. 모든 event가 여기를 통과.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata: { name: default, namespace: argo-events }
spec:
  nats:
    native:
      replicas: 3
      auth: token
```

### EventSource
외부 시스템에서 event 수집:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata: { name: webhook, namespace: argo-events }
spec:
  service:
    ports:
      - port: 12000
        targetPort: 12000
  webhook:
    build-trigger:
      port: "12000"
      endpoint: /build
      method: POST
```

source 종류: webhook, github, gitlab, s3 (MinIO), kafka, redis, sqs, sns, gcs, pubsub, calendar, cron, file, ...

### Sensor
event를 받아 조건 평가 후 trigger:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata: { name: webhook-sensor }
spec:
  dependencies:
    - name: webhook-dep
      eventSourceName: webhook
      eventName: build-trigger
  triggers:
    - template:
        name: workflow-trigger
        k8s:
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: webhook-build-
              spec:
                entrypoint: main
                templates:
                  - name: main
                    container:
                      image: alpine
                      command: [echo]
                      args: ["triggered from webhook"]
```

## 3. GitHub Source 예시

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata: { name: github }
spec:
  github:
    mycorp-myapp:
      repositories:
        - owner: mycorp
          names: [myapp]
      webhook:
        endpoint: /push
        port: "12000"
      events:
        - push
        - pull_request
      apiToken:
        name: github-token
        key: token
      insecure: false
      active: true
```

push event 시 자동 workflow trigger. CI 자동화 패턴.

## 4. Filter / Conditional Trigger

```yaml
dependencies:
  - name: push-dep
    eventSourceName: github
    eventName: mycorp-myapp
    filters:
      data:
        - path: body.ref
          type: string
          value: ["refs/heads/main"]
```

main brunch push만 trigger.

## 5. Parameters from Event

```yaml
triggers:
  - template:
      k8s:
        operation: create
        source: ...
        parameters:
          - src:
              dependencyName: push-dep
              dataKey: body.head_commit.id
            dest: spec.arguments.parameters.0.value
```

git commit SHA를 workflow parameter로 전달.

## 6. 빅테크 패턴 — Self-service Trigger

플랫폼팀이 EventSource·Sensor 표준 셋 운영. 앱팀은 자기 repo에 webhook URL 등록만.

## 7. ArgoCD와 결합

EventSource로 git push 받음 → Sensor가 ArgoCD Application sync trigger:

```yaml
triggers:
  - template:
      argoWorkflow:
        operation: submit
        source: ...
      http:
        url: https://argocd.mycorp.com/api/v1/applications/{{.Input.appName}}/sync
        method: POST
        headers:
          Authorization: Bearer {{.Secret.argocd-token}}
```

## 8. 운영 함정 5선

1. EventBus replicas 1 → 단일 장애.
2. webhook secret 없음 → 외부 누구나 trigger.
3. filter 누락 → 모든 commit이 trigger.
4. retry policy 누락 → 일시 장애에 event 손실.
5. event log 미보존 → 디버깅 지옥.

## 9. 실습

```bash
# 1. Argo Events 설치
# 2. webhook EventSource + Sensor → echo workflow
# 3. curl로 trigger 테스트
# 4. GitHub Source 설정 + 본인 repo push trigger
# 5. filter로 main branch만
# 6. parameter로 commit SHA 전달
```

## 10. 자가평가 퀴즈

### Q1. 3 컴포넌트?
1. **EventBus, EventSource, Sensor**
2. Workflow, Step, DAG
3. UI
4. 무관

**정답: 1.**

### Q2. EventBus 기반 기술?
1. **NATS Streaming**
2. Kafka 내장
3. Redis
4. 무관

**정답: 1.**

### Q3. Filter 가치?
1. **특정 조건의 event만 trigger**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q4. webhook secret 누락의 위험?
1. **외부 누구나 trigger 가능 → 침해**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. self-service trigger 패턴?
1. **플랫폼팀 표준 + 앱팀 webhook URL 등록**
2. UI
3. 무관
4. 비용

**정답: 1.**

## 11. 다음 주차

[Week 6: Tekton 기초]에서 Task/Pipeline/PipelineRun을 다룹니다.
