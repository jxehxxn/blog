---
layout: post
title: "Distributed Task Processing with Celery & Redis: Week 2 - Celery Fundamentals & Architecture"
---

"Celery가 정확히 무슨 일을 하는 건가요?"
단순히 함수를 뒤에서 실행해주는 라이브러리가 아닙니다. Celery는 **분산 시스템 프레임워크**입니다.

오늘은 Celery를 구성하는 4대 요소(Client, Broker, Worker, Result Backend)가 어떻게 유기적으로 움직이는지 해부해 봅니다. 이 그림이 머릿속에 없으면 트러블슈팅을 할 수 없습니다.

---

## 1. The Architecture of Celery

### 1) Client (Producer)
작업을 요청하는 주체입니다. Django/Flask 웹 서버가 될 수도 있고, 스크립트일 수도 있습니다.
*   `task.delay()`를 호출하면, 작업을 JSON으로 직렬화(Serialization)해서 Broker에게 던집니다.

### 2) Broker (Dispatcher)
우체국입니다. Client가 던진 작업을 받아서, 적절한 Worker가 가져갈 때까지 큐(Queue)에 보관합니다.
*   Redis, RabbitMQ, SQS 등을 사용합니다.

### 3) Worker (Consumer)
우체부 혹은 일꾼입니다. Broker를 계속 감시하다가, 작업이 들어오면 가져와서 실행합니다.
*   `celery worker` 프로세스입니다.
*   작업을 수행하고 성공/실패 여부를 판단합니다.

### 4) Result Backend (Storage)
작업의 결과(Return Value)나 상태(Started, Success, Failure)를 저장하는 곳입니다.
*   Client가 `AsyncResult` 객체를 통해 "그 작업 다 됐어?"라고 물어볼 때, 여기를 조회합니다.
*   Broker와 같은 Redis를 쓸 수도 있고, DB(PostgreSQL)를 쓸 수도 있습니다.

---

## 2. Serialization (직렬화)

Client와 Worker는 서로 다른 프로세스, 심지어 서로 다른 서버에 있을 수 있습니다.
따라서 Python 객체를 그대로 전달할 수 없습니다. **JSON, Pickle, YAML** 등으로 변환해야 합니다.

*   **Pickle:** Python 전용. 모든 객체를 변환 가능하지만 **보안 위험**이 큼. (신뢰할 수 없는 데이터 역직렬화 시 해킹 위험)
*   **JSON:** 표준. 안전함. 하지만 `datetime`이나 커스텀 객체 변환이 까다로움. **Celery 4.0부터 기본값.**

**Best Practice:**
항상 **JSON**을 사용하고, Task의 인자로는 **복잡한 객체(Django Model 등) 대신 ID(Primary Key)**만 넘기세요.
`process_user(user_obj)` (X) -> `process_user(user_id)` (O)

---

## 3. Worker Concurrency

"워커 하나가 동시에 몇 개의 작업을 처리할 수 있나요?"

기본적으로 Celery Worker는 **Prefork** 방식을 사용합니다.
부모 프로세스가 있고, CPU 코어 수만큼 자식 프로세스를 미리 띄워놓습니다.

*   `--concurrency=4`: 자식 프로세스 4개를 띄움. 동시에 4개 작업 처리 가능.
*   **I/O Bound 작업:** (네트워크 요청 등) 코어 수보다 많이 띄워도 됨. (Threads/Gevent/Eventlet 사용 권장)
*   **CPU Bound 작업:** (이미지 처리 등) 코어 수만큼만 띄우는 게 좋음.

---

## 🛠️ Lab: Configuring Celery

`celeryconfig.py`를 분리하여 설정을 관리하는 법을 배웁니다.

1.  **Project Structure:**
    ```
    myproject/
    ├── celery.py       # Celery 인스턴스 정의
    ├── tasks.py        # Task 정의
    ├── config.py       # 설정 파일
    └── run.py          # Client 스크립트
    ```

2.  **config.py:**
    ```python
    broker_url = 'redis://localhost:6379/0'
    result_backend = 'redis://localhost:6379/1'
    task_serializer = 'json'
    result_serializer = 'json'
    accept_content = ['json']
    enable_utc = True
    ```

3.  **celery.py:**
    ```python
    from celery import Celery
    app = Celery('myproject')
    app.config_from_object('config')
    ```

4.  **Worker 실행:** `celery -A myproject worker -l info`

---

## 📝 2주차 과제: Result Backend Check

**목표:** Result Backend에 실제로 데이터가 어떻게 저장되는지 확인하세요.

1.  `add` 태스크를 실행하고 `task_id`를 받으세요.
2.  `redis-cli`를 사용하여 Result Backend DB(`1`번 DB)를 조회하세요. (`SELECT 1`, `KEYS *`)
3.  `celery-task-meta-{task_id}` 키의 값을 `GET` 하여 JSON 내용을 캡처하세요.
    *   상태(`status`), 결과(`result`), 완료 시간 등이 보이는지 확인.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Celery에서 Task의 인자로 ORM 객체(예: Django User 모델)를 직접 넘기는 것이 권장되지 않는 이유는? (직렬화 문제 + 데이터 정합성 문제 - 큐에 있는 동안 DB 데이터가 변할 수 있음)
2.  **Q2.** 보안을 위해 Pickle 대신 권장되는 직렬화 포맷은? (JSON)
3.  **Q3.** Worker가 Broker에게 작업을 가져왔음을 알리는 신호를 무엇이라 하나요? (ACK - Acknowledgement)

---

다음 주, Celery 사용자가 가장 많이 저지르는 실수인 **"멱등성(Idempotency)"** 문제를 다룹니다.
작업이 두 번 실행되어도 괜찮게 만드는 법, 이것이 분산 시스템의 핵심입니다.

**Serialized success.**
