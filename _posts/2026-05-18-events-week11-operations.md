---
layout: post
title: "Events Week 11: 운영 + Scale"
date: 2026-05-18 23:00:00 +0900
categories: argo-events platform senior-series
---

## 1. EventBus HA
3 replica + persistence.

## 2. EventSource scale
type별 다름:
- webhook: HPA로 replica.
- Kafka: partition 수만큼.
- cron: 1개로 충분.

## 3. Sensor scale
복잡한 sensor는 메모리 부담. HPA.

## 4. Monitoring
```promql
argoevents_sensor_triggered_total
argoevents_eventsource_received_total
argoevents_eventsource_failed_total
```

## 5. Backpressure
EventBus가 메시지 쌓이면 source가 receive 늦춤. JetStream의 ack 설계.

## 6. Upgrade
EventBus는 stateful → careful upgrade.

## 7. 함정
- replica 1.
- backpressure 무시.
- 모니터링 부재.
- upgrade 시 message 손실.

## 8. 자가평가
### Q1. EventBus HA? **3 replica + persistence**. 정답 1.
### Q2. webhook scale? **HPA**. 정답 1.
### Q3. Kafka scale? **partition 수**. 정답 1.
### Q4. backpressure? **JetStream ack**. 정답 1.
### Q5. EventBus upgrade? **stateful → careful**. 정답 1.

## 다음
[Week 12: 캡스톤].
