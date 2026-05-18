---
layout: post
title: "Events Week 1: 오리엔테이션 — Event-driven K8s"
date: 2026-05-18 23:00:00 +0900
categories: argo-events platform senior-series
---

## 학습 목표
- Event-driven의 가치.
- Argo Events 3 컴포넌트.
- 다른 도구 비교.

## 1. 비유 — "알림 인터컴 + 자동 반응 로봇"

공장 알림(EventSource) → 인터컴(EventBus) → 반응 로봇(Sensor) → 작업(Trigger).

## 2. 3 컴포넌트
- EventBus: NATS Streaming.
- EventSource: 외부 system → bus 게시.
- Sensor: bus 구독 + 조건 평가 + trigger.

## 3. 대안 비교
- Knative Eventing: CloudEvents 표준, K8s native, 다양한 source.
- Kafka 자체: 직접 consumer 작성 (대규모, 표준).
- Argo Events: K8s 친화 + Argo Workflow 통합 강력.

## 4. 사용 사례
- GitHub push → CI workflow.
- S3 upload → 데이터 처리.
- Kafka message → 알람.
- Cron → 정기 작업.

## 5. 자가평가
### Q1. 3 컴포넌트? **EventBus/EventSource/Sensor**. 정답 1.
### Q2. EventBus 기술? **NATS Streaming**. 정답 1.
### Q3. Argo Events vs Knative? **Argo Workflow 통합 강력**. 정답 1.
### Q4. 사용 사례? **GitHub/S3/Kafka/cron**. 정답 1.
### Q5. 시작 단계? **CI trigger 가장 흔함**. 정답 1.

## 다음
[Week 2: EventBus].
