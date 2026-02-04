---
layout: post
title: "Backend Architecture Mastery: Week 7 - Distributed Task Processing (Celery)"
---

지난주까지 RabbitMQ와 `aio_pika`를 써서 바닥부터(Raw level) 큐 시스템을 만들었습니다. 공부하기엔 최고지만, 실무에서 매번 Retry, Serialization, Connection Pool을 직접 구현하는 건 비효율적입니다.

그래서 등장했습니다. Python 분산 작업의 사실상 표준(De facto standard), **Celery**입니다.

---

## 1. What is Celery?

Celery는 "Distributed Task Queue" 프레임워크입니다.
*   **Broker:** 메시지 운반책 (RabbitMQ, Redis)
*   **Worker:** 작업을 처리하는 일꾼 (Celery가 관리)
*   **Backend:** 결과를 저장하는 곳 (Redis, DB)

우리가 5주차에 짰던 수백 줄의 코드가 Celery를 쓰면 단 5줄이 됩니다.

```python
from celery import Celery

app = Celery('tasks', broker='amqp://guest@localhost//')

@app.task
def add(x, y):
    return x + y
```

---

## 2. Celery Architecture & Features

### 강력한 기능들
1.  **Chaining (Canvas):** 작업 A가 끝나면 B를 하고, 그 결과로 C를 해라. (`chain`)
2.  **Scheduling (Beat):** 매일 아침 9시에 리포트를 발송해라. (Crontab 지원)
3.  **Monitoring (Flower):** 현재 처리 중인 작업, 실패율, 워커 상태를 웹 UI로 보여줍니다.

### 단점 (The Dark Side)
*   **너무 무겁습니다:** 설정이 복잡하고 배우는 데 시간이 걸립니다.
*   **디버깅 지옥:** 비동기로 돌다가 어디선가 죽으면, 로그 찾기가 셜록 홈즈급 추리력을 요구합니다.
*   **Windows 지원 미비:** 리눅스/맥 환경이 강제됩니다.

---

## 3. Dramatiq: The Modern Challenger

Celery가 너무 복잡해서 나온 대안이 **Dramatiq**입니다.
*   **장점:** 설정이 거의 필요 없음(Zero configuration), RabbitMQ 친화적, 메시지 우선순위 지원이 더 직관적.
*   **단점:** Celery만큼 생태계가 크지 않음 (Django 연동 등).

하지만 대부분의 기업은 레거시 때문에, 혹은 안정성 때문에 여전히 Celery를 씁니다. 우리도 Celery를 배웁니다.

---

## 🛠️ Lab: Integrating Celery with FastAPI

FastAPI와 Celery는 프로세스가 다릅니다. 이 둘을 어떻게 연동할까요?

### 구조
*   `main.py` (FastAPI): 작업을 `delay()` 메서드로 Celery에 던짐.
*   `tasks.py` (Celery): 실제 로직이 들어있는 곳.

```python
# tasks.py
from celery import Celery

celery_app = Celery(
    "worker", 
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/1"
)

@celery_app.task(name="create_thumbnail")
def create_thumbnail(image_path):
    # 이미지 처리...
    return f"{image_path}_thumb.jpg"
```

```python
# main.py
from fastapi import FastAPI
from tasks import create_thumbnail

app = FastAPI()

@app.post("/upload")
def upload_image(file_path: str):
    # .delay()를 호출하면 비동기로 큐에 들어갑니다.
    task = create_thumbnail.delay(file_path)
    return {"task_id": task.id}
```

---

## 📝 7주차 과제: Email Campaign System

**목표:** 대량 메일 발송 시스템을 Celery로 구축하세요.

1.  **RabbitMQ**를 브로커로 사용하세요.
2.  **Task 1:** `send_email(user_email)` - 1초 소요. 랜덤하게 실패하도록 설정 후 `autoretry_for` 옵션으로 자동 재시도 적용.
3.  **Task 2 (Chain):** 이메일 발송이 성공하면 `log_email_success(user_email)` 태스크가 이어서 실행되도록 체이닝(`chain`) 하세요.
4.  **Flower**를 띄워서 작업 진행 상황을 캡처하여 제출하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Celery에서 주기적인 작업(예: 매일 자정 실행)을 담당하는 컴포넌트의 이름은? (Celery Beat)
2.  **Q2.** Celery 워커가 작업을 가져갈 때, 미리 여러 개를 가져가서 쌓아두는 설정을 무엇이라 하나요? (Prefetch Multiplier)
3.  **Q3.** Task의 결과를 저장하고 조회하기 위해 설정해야 하는 컴포넌트는? (Result Backend)

---

다음 주에는 큐(Queue)를 넘어 **스트림(Stream)**의 세계로 갑니다. Kafka와 Event-Driven Architecture에 대해 알아봅니다.

**Queue it up.**
