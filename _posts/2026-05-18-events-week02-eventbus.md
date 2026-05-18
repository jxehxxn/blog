---
layout: post
title: "Events Week 2: EventBus (NATS Streaming)"
date: 2026-05-18 23:00:00 +0900
categories: argo-events platform senior-series
---

## 1. EventBus CR

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata: { name: default, namespace: argo-events }
spec:
  nats:
    native:
      replicas: 3
      auth: token
      persistence:
        storageClassName: ssd
        accessMode: ReadWriteOnce
        volumeSize: 10Gi
```

3 replica = quorum.

## 2. 옵션

- `native`: Argo가 NATS 직접 띄움.
- `exotic`: 외부 NATS cluster 연결.

빅테크: external NATS 운영 (전용 cluster).

## 3. JetStream (NATS 새 streaming)

NATS Streaming deprecated → JetStream:
```yaml
spec:
  jetstream:
    replicas: 3
    version: 2.10
```

durable, exactly-once, at-least-once 보장.

## 4. 운영
- replica 3 + persistent storage.
- monitoring (NATS exporter).
- backup (메시지 자체는 일시적).

## 5. 함정
- replica 1 → 단일 장애.
- persistence 없음 → 재시작 시 메시지 손실.

## 6. 자가평가
### Q1. 권장 replica? **3 (quorum)**. 정답 1.
### Q2. JetStream? **NATS 새 streaming (durable)**. 정답 1.
### Q3. exotic? **외부 NATS 연결**. 정답 1.
### Q4. 단일 replica? **장애 위험**. 정답 1.
### Q5. monitoring? **NATS exporter**. 정답 1.

## 다음
[Week 3: EventSource].
