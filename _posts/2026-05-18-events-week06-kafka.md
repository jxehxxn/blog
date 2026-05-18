---
layout: post
title: "Events Week 6: Kafka + Webhook Source"
date: 2026-05-18 23:00:00 +0900
categories: argo-events platform senior-series
---

## Kafka Source

```yaml
spec:
  kafka:
    payments-events:
      url: kafka.svc:9092
      topic: payments
      partition: "0"
      consumerGroup: { groupName: argo-events-payments }
      jsonBody: true
      tls:
        caCertSecret: { name: kafka-ca, key: ca.crt }
        clientCertSecret: { name: kafka-client, key: tls.crt }
```

Kafka topic → bus.

## Webhook 추가 옵션

```yaml
webhook:
  trigger-name:
    port: "12000"
    endpoint: /trigger
    method: POST
    authSecret:
      name: webhook-auth
      key: token
```

bearer token 검증.

## SSL

webhook https:
```yaml
spec:
  webhook:
    ...
serverCertSecret:
  name: webhook-tls
  key: tls.crt
```

## 자가평가
### Q1. Kafka source? **topic → bus**. 정답 1.
### Q2. consumerGroup? **소비 offset 관리**. 정답 1.
### Q3. webhook authSecret? **bearer 검증**. 정답 1.
### Q4. webhook TLS? **serverCertSecret**. 정답 1.
### Q5. jsonBody? **JSON parsing**. 정답 1.

## 다음
[Week 7: S3 / SQS / PubSub].
