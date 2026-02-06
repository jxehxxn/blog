---
layout: post
title: "Distributed Task Processing with Celery & Redis: Week 8 - Monitoring & Management (Flower, Prometheus)"
---

"워커가 죽었나요?" "작업이 몇 개나 밀려있죠?"
터미널에서 `redis-cli LLEN`만 치고 있을 순 없습니다.
GUI 대시보드가 필요합니다. Celery의 영원한 단짝, **Flower**를 소개합니다.

---

## 1. Flower: The Celery Dashboard

Flower는 웹 기반의 Celery 모니터링 도구입니다.
*   **실시간 작업 현황:** 성공, 실패, 재시도 횟수.
*   **워커 상태:** 어떤 워커가 온라인인지, 로드(Load)가 얼마나 걸렸는지.
*   **제어 기능:** 실행 중인 작업 취소(Revoke), 워커 재시작, 속도 제한(Rate Limit) 변경.

### Installation
```bash
pip install flower
celery -A myproj flower
```
`localhost:5555`로 접속하면 신세계가 열립니다.

---

## 2. Prometheus & Grafana Integration

Flower는 "지금"을 보는 데는 좋지만, "어제 점심시간에 작업이 얼마나 몰렸는지" 같은 **시계열 데이터(Time-series)** 분석에는 약합니다.
Celery 데이터를 Prometheus로 수출(Export)해야 합니다.

### celery-exporter
`celery-exporter`라는 별도 프로세스를 띄우면, Redis Broker를 조회하여 메트릭을 수집합니다.
*   `celery_queue_length`: 큐 길이.
*   `celery_active_tasks`: 실행 중인 작업 수.

이걸 Grafana로 시각화하면 "큐가 폭발하기 직전"을 예측하고 오토스케일링을 할 수 있습니다.

---

## 3. Celery Events

Flower는 어떻게 실시간으로 상태를 알까요?
Celery Worker는 작업을 처리할 때마다 **이벤트(Event)**를 발생시킵니다.
*   `task-received`
*   `task-started`
*   `task-succeeded`

Flower는 이 이벤트를 구독(Subscribe)하고 있다가 화면을 갱신합니다.
단, 이벤트 발생은 **오버헤드**가 있습니다. 프로덕션에서 너무 많은 작업이 몰리면 (`-E` 옵션) 성능 저하가 올 수 있으니 주의해야 합니다.

---

## 🛠️ Lab: Monitoring Stack

Flower와 Prometheus를 연동해 봅니다.

1.  **Flower 실행:** `celery -A tasks flower`
2.  **부하 발생:** `for i in range(100): add.delay(i, i)` 스크립트 실행.
3.  **Flower 확인:**
    *   Processed 탭에서 그래프가 올라가는지 확인.
    *   Tasks 탭에서 작업 UUID와 인자값 확인.
4.  **제어:** 실행 중인(Long running) 작업을 찾아서 'Terminate' 버튼 눌러보기.

---

## 📝 8주차 과제: Alerting Rule

**목표:** Prometheus AlertManager 규칙을 작성하세요. (가상)

1.  **상황:** `default` 큐에 대기 중인 작업(`celery_queue_length`)이 1000개를 넘고, 이 상태가 5분 이상 지속되면 슬랙으로 알람을 보내야 합니다.
2.  이 로직을 Prometheus Rule YAML 포맷으로 작성하여 제출하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Flower 대시보드에서 제공하는 기능 중, 실행 중인 Task를 강제로 중단시키는 기능은? (Revoke / Terminate)
2.  **Q2.** Celery Worker가 모니터링 도구(Flower)를 위해 상태 변화 이벤트를 보내도록 켜는 옵션 플래그는? (`-E` 또는 `--events`)
3.  **Q3.** Flower는 기본적으로 몇 번 포트에서 실행되나요? (5555)

---

다음 주, "작업이 너무 느려요."
워커의 성능을 극한으로 끌어올리는 **Tuning** 기법을 배웁니다. `prefetch_multiplier`의 비밀을 밝힙니다.

**Watch the watchers.**
