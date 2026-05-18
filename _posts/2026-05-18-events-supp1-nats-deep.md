---
layout: post
title: "Events 보충 1: NATS / JetStream Deep"
date: 2026-05-18 23:00:00 +0900
categories: argo-events nats supplement
---

## NATS Core
pub/sub, 무지속 (at-most-once).

## JetStream
durable, at-least-once, exactly-once.

## Streams
topic 그룹. retention, replication.

## Consumers
- push consumer.
- pull consumer.

## Operator
HA + persistence.

## 결론
JetStream이 빅테크 표준.
