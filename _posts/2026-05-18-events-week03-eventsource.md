---
layout: post
title: "Events Week 3: EventSource 종류"
date: 2026-05-18 23:00:00 +0900
categories: argo-events platform senior-series
---

## 1. EventSource CR

```yaml
spec:
  webhook:
    name-of-source:
      port: "12000"
      endpoint: /push
      method: POST
```

각 type별 spec field.

## 2. Type 카탈로그
- webhook
- github/gitlab/bitbucket
- s3/minio (object created/deleted)
- kafka
- sqs / sns / pubsub
- redis
- nats
- amqp (RabbitMQ)
- mqtt
- file (filesystem watch)
- calendar (cron)
- emitter, generic (custom)

## 3. webhook 예시
```yaml
spec:
  service:
    ports: [{port: 12000, targetPort: 12000}]
  webhook:
    deploy-trigger:
      port: "12000"
      endpoint: /deploy
      method: POST
```

Service로 외부 노출.

## 4. file source
파일 시스템 watch (Kubernetes 노드의 디렉토리):
```yaml
file:
  example:
    eventType: CREATE
    watchPathConfig:
      directory: /data
      path: "*.csv"
```

## 5. calendar (cron)
```yaml
calendar:
  daily:
    schedule: "0 2 * * *"
```

CronWorkflow 대안. 더 풍부한 trigger 가능.

## 6. 자가평가
### Q1. EventSource 책임? **외부 system → bus**. 정답 1.
### Q2. webhook 노출? **Service**. 정답 1.
### Q3. cron 대체? **calendar source**. 정답 1.
### Q4. s3 source trigger? **object 생성/삭제**. 정답 1.
### Q5. 다양성? **15+ type**. 정답 1.

## 다음
[Week 4: Sensor + Trigger].
