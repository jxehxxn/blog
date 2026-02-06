---
layout: post
title: "Distributed Task Processing with Celery & Redis: Week 10 - Advanced Routing & Queues"
---

모든 작업을 `default` 큐 하나에 넣으면 문제가 생깁니다.
"뉴스레터 10만 통 발송" 작업이 큐를 꽉 채우면, 그 뒤에 들어온 "비밀번호 찾기 이메일"은 10시간 뒤에 발송됩니다. 사용자는 떠나겠죠.

중요한 작업은 **전용 도로(Queue)**를 내줘야 합니다. 이것이 **라우팅(Routing)**입니다.

---

## 1. Queue Definition

먼저 큐를 정의해야 합니다.

```python
from kombu import Queue

app.conf.task_queues = (
    Queue('default', routing_key='default'),
    Queue('high_priority', routing_key='high.#'),
    Queue('low_priority', routing_key='low.#'),
)
```

---

## 2. Automatic Routing

어떤 Task가 어떤 큐로 갈지 규칙을 정합니다.

```python
app.conf.task_routes = {
    'tasks.send_password_reset': {'queue': 'high_priority'},
    'tasks.send_newsletter': {'queue': 'low_priority'},
    'tasks.*': {'queue': 'default'}, # 나머지 전부
}
```

이제 `send_password_reset.delay()`를 하면 알아서 `high_priority` 큐로 들어갑니다.

---

## 3. Worker Assignment

큐를 나누는 것만으론 부족합니다. 워커에게 "너는 이 큐만 처리해"라고 지정해야 합니다.

*   **Worker A (VIP):** `celery worker -Q high_priority`
    *   오직 중요한 작업만 처리합니다. 일반 작업이 100만 개 쌓여도 신경 안 씁니다.
*   **Worker B (General):** `celery worker -Q default,low_priority`
    *   일반 작업과 낮은 우선순위 작업을 처리합니다.

---

## 4. Manual Routing (`apply_async`)

코드 실행 시점에 동적으로 큐를 바꿀 수도 있습니다.

```python
# 이번만 특별히 빨리 처리해줘
generate_report.apply_async(args=[user_id], queue='high_priority')
```

---

## 🛠️ Lab: Queue Separation

1.  **설정:** `high`, `default` 두 개의 큐를 정의합니다.
2.  **Task:** `urgent_task`와 `normal_task`를 만듭니다.
3.  **라우팅:** `urgent_task` -> `high`, `normal_task` -> `default`.
4.  **실행:**
    *   터미널 1: `celery worker -Q high` (VIP 워커)
    *   터미널 2: `celery worker -Q default` (일반 워커)
5.  **테스트:** `normal_task`를 100개 던져서 일반 워커를 바쁘게 만듭니다. 그 사이에 `urgent_task`를 던지면 VIP 워커가 즉시 채가는지 확인합니다.

---

## 📝 10주차 과제: Multi-Tenant Routing

**목표:** SaaS 서비스에서 '유료 플랜' 고객과 '무료 플랜' 고객의 처리 속도를 다르게 하세요.

1.  사용자 ID를 받아 플랜을 조회하는 로직은 생략(가정)합니다.
2.  `process_data(user_id)` Task를 호출할 때, 유료 회원이면 `premium` 큐로, 아니면 `free` 큐로 보내는 라우터(Router Class)를 작성하거나 호출부 로직을 작성하세요.
3.  `premium` 큐는 워커 5대가, `free` 큐는 워커 1대가 처리하도록 구성도를 그리세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 특정 Task를 항상 특정 큐로 보내도록 설정하는 Celery 설정 항목은? (task_routes)
2.  **Q2.** 워커 프로세스를 실행할 때, 이 워커가 구독할 큐 목록을 지정하는 옵션은? (`-Q` 또는 `--queues`)
3.  **Q3.** 런타임에 동적으로 큐를 지정하여 Task를 발행하고 싶을 때 `delay()` 대신 사용해야 하는 메서드는? (`apply_async(queue='...')`)

---

다음 주, 코드가 잘 돌아가는지 어떻게 확신하나요?
비동기 코드는 테스트하기 어렵습니다. Mocking과 Eager Mode를 이용한 **Test Strategy**를 배웁니다.

**Priority first.**
