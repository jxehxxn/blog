---
layout: post
title: "Events Week 8: Trigger 종류 깊게"
date: 2026-05-18 23:00:00 +0900
categories: argo-events platform senior-series
---

## k8s Trigger

```yaml
triggers:
  - template:
      k8s:
        operation: create  # 또는 update/patch/delete
        source:
          resource:
            apiVersion: ...
            kind: ...
```

## argoWorkflow

```yaml
triggers:
  - template:
      argoWorkflow:
        operation: submit  # submit/resume/retry/resubmit/terminate
        source:
          resource: { kind: Workflow, ... }
```

## http

```yaml
triggers:
  - template:
      http:
        url: https://api.mycorp.com/notify
        method: POST
        headers:
          - { name: Authorization, value: "Bearer $token" }
        payload:
          - src: { dependencyName: dep, dataKey: body }
            dest: data
```

SOAR/Slack 등 외부 webhook.

## lambda

```yaml
triggers:
  - template:
      awsLambda:
        functionName: my-function
        region: us-east-1
        accessKey: ...
```

## kafka

```yaml
triggers:
  - template:
      kafka:
        url: kafka:9092
        topic: alerts
        partition: 0
        payload: [...]
```

## slack

```yaml
triggers:
  - template:
      slack:
        slackToken: { name: slack-token, key: token }
        channel: "#alerts"
        message: "build started"
```

## 자가평가
### Q1. k8s operation? **create/update/patch/delete**. 정답 1.
### Q2. argoWorkflow operation? **submit/resume/retry**. 정답 1.
### Q3. http trigger 용도? **외부 webhook (SOAR/Slack)**. 정답 1.
### Q4. lambda? **AWS Lambda invoke**. 정답 1.
### Q5. slack? **#channel notification**. 정답 1.

## 다음
[Week 9: Filter + parameter].
