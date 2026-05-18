---
layout: post
title: "Events Week 7: S3 / SQS / PubSub Source"
date: 2026-05-18 23:00:00 +0900
categories: argo-events platform senior-series
---

## S3 (MinIO 호환)

```yaml
spec:
  minio:
    payments-uploads:
      bucket: { name: payments-uploads }
      events: ["s3:ObjectCreated:*"]
      filter: { prefix: "txn/", suffix: ".csv" }
      endpoint: s3.amazonaws.com
      accessKey: { name: aws-creds, key: access }
      secretKey: { name: aws-creds, key: secret }
```

S3 ObjectCreated → bus.

## SQS

```yaml
spec:
  sqs:
    work-queue:
      queue: my-queue
      waitTimeSeconds: 20
      region: us-east-1
      accessKey: ...
      secretKey: ...
```

SQS message → bus.

## GCP PubSub

```yaml
spec:
  pubSub:
    incoming:
      projectID: my-project
      topic: incoming
      subscriptionID: argo-sub
      credentialSecret: { name: gcp-creds, key: key.json }
```

## 사용 사례
- S3 upload → 데이터 처리 workflow.
- SQS work queue → distributed job.
- PubSub event → 알람.

## 자가평가
### Q1. S3 trigger? **ObjectCreated**. 정답 1.
### Q2. SQS waitTimeSeconds? **long polling**. 정답 1.
### Q3. PubSub credential? **secret (key.json)**. 정답 1.
### Q4. MinIO 호환? **S3 API 표준**. 정답 1.
### Q5. filter? **prefix/suffix**. 정답 1.

## 다음
[Week 8: Trigger 종류].
