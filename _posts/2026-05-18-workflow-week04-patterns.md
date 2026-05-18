---
layout: post
title: "Workflow Week 4: 패턴 — DAG, Retry, Conditionals, Suspend, Exit Handler"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series
---

## 학습 목표

- DAG 복잡 패턴(fan-out, fan-in, conditional branch)을 작성.
- Retry/backoff 정책 운영.
- Suspend로 사람 승인 게이트.
- Exit handler로 항상 실행될 cleanup.

## 1. Fan-out / Fan-in

비유: 한 상자(input)를 100명이 동시에 처리 → 다시 합쳐 출력.

```yaml
- name: fan
  steps:
    - - name: split
        template: split
    - - name: process
        template: process
        arguments:
          parameters:
            - { name: item, value: "{{item}}" }
        withParam: "{{steps.split.outputs.result}}"   # JSON array
    - - name: merge
        template: merge
        arguments:
          artifacts:
            - { name: all, from: "{{steps.process.outputs.artifacts.x}}" }
```

`withParam` (JSON array) 또는 `withItems` (인라인 list) 또는 `withSequence` (숫자 범위).

## 2. Conditional

```yaml
- name: pipeline
  steps:
    - - name: test
        template: test
    - - name: deploy-prod
        template: deploy-prod
        when: "{{steps.test.outputs.parameters.result}} == success && {{workflow.parameters.env}} == prod"
      - name: deploy-stage
        template: deploy-stage
        when: "{{workflow.parameters.env}} == stage"
```

govaluate expression으로 평가.

## 3. Retry & Backoff

```yaml
- name: flaky
  retryStrategy:
    limit: 5
    retryPolicy: OnFailure
    backoff:
      duration: "1m"
      factor: 2
      maxDuration: "30m"
    expression: "lastRetry.exitCode != 1"   # exit 1은 재시도 안 함
  container: ...
```

운영 권장: 외부 API 호출은 항상 retry. 내부 비즈니스 로직 실패는 신중.

## 4. Timeout

```yaml
- name: slow
  activeDeadlineSeconds: 600   # 10분
  container: ...
```

또는 workflow 전체:
```yaml
spec:
  activeDeadlineSeconds: 7200
```

## 5. Suspend (Approval Gate)

```yaml
- name: approve
  suspend: {}   # 무제한 대기, 사람이 resume

# 또는
- name: wait-1h
  suspend:
    duration: "1h"
```

resume:
```bash
argo resume my-wf
```

ArgoCD UI에서 클릭으로 resume도 가능. 빅테크 패턴: prod 배포 전 매니저 승인 단계.

## 6. Exit Handler — 항상 실행

비유: 시험 끝나면 답안지 제출 (성공/실패 무관).

```yaml
spec:
  onExit: cleanup
  templates:
    - name: cleanup
      container:
        image: alpine
        command: [sh, -c]
        args:
          - |
            echo "workflow ended with status: {{workflow.status}}"
            # notify slack, cleanup resources etc.
```

`{{workflow.status}}` 으로 success/failed 분기 가능.

## 7. ContinueOn (실패해도 계속)

```yaml
- name: a
  template: a
  continueOn:
    failed: true
```

a 실패해도 다음 step 진행. 비핵심 metric 수집 같은 곳에 유용.

## 8. ParallelStep 제한

```yaml
spec:
  parallelism: 10   # 동시 step 최대 10
```

큰 workflow에서 controller·노드 부하 조절.

## 9. Synchronization (Mutex / Semaphore)

같은 자원에 동시 접근 차단:

```yaml
spec:
  synchronization:
    mutex:
      name: deploy-prod
```

prod 배포 동시 발생 차단. configmap 기반 semaphore도 있음.

## 10. Workflow Template (재사용)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata: { name: build-and-test }
spec:
  templates: [...]
```

여러 workflow가 reference. 사내 표준 빌드 절차를 한 곳에.

## 11. ClusterWorkflowTemplate

WorkflowTemplate은 namespace 단위. Cluster 단위로 공유하려면 `ClusterWorkflowTemplate`.

## 12. CronWorkflow

cron schedule로 자동 실행:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: CronWorkflow
metadata: { name: nightly-etl }
spec:
  schedule: "0 2 * * *"
  workflowSpec:
    entrypoint: main
    templates: [...]
```

## 13. 운영 함정 5선

1. Retry limit 너무 큼 → 실패 알람 지연.
2. activeDeadline 누락 → 무한 실행.
3. Exit handler 누락 → 실패 알람 없음.
4. parallelism 무제한 → controller 폭주.
5. WorkflowTemplate 안 쓰고 yaml 복사 → drift.

## 14. 실습

```bash
# 1. withItems로 fan-out 5개
# 2. conditional로 env=prod 만 deploy
# 3. retry 3회 + backoff
# 4. suspend로 사람 승인 게이트
# 5. exit handler로 Slack 알람
# 6. WorkflowTemplate 만들어 여러 workflow에서 사용
```

## 15. 자가평가 퀴즈

### Q1. fan-out의 도구?
1. **withParam / withItems / withSequence**
2. exit handler
3. mutex
4. 무관

**정답: 1.**

### Q2. Retry 권장 적용 대상?
1. **외부 API 호출 등 transient 실패**
2. 비즈니스 로직 실패
3. 모든 경우
4. 무관

**정답: 1.**

### Q3. Exit handler 가치?
1. **성공/실패 무관 cleanup·알람**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q4. mutex 사용처?
1. **같은 자원 동시 접근 차단**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q5. WorkflowTemplate 가치?
1. **재사용 가능한 사내 표준 절차 정의**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 16. 다음 주차

[Week 5: Argo Events]에서 외부 이벤트(웹훅, S3, Kafka) → workflow trigger를 다룹니다.
