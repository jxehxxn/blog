---
layout: post
title: "Distributed Task Processing with Celery & Redis: Week 1 - Orientation"
---

안녕하세요, 백엔드 엔지니어 여러분.

웹 개발을 하다 보면 반드시 마주치는 문제가 있습니다.
"사용자가 '리포트 생성' 버튼을 눌렀는데, 생성이 1분 걸려요. 사용자는 1분 동안 로딩 바만 보고 있어야 하나요?"
"이메일 1만 통을 보내야 하는데, 서버가 멈췄어요."

이것은 **동기(Synchronous)** 처리의 한계입니다. 서버가 할 일을 다 마칠 때까지 클라이언트를 붙잡고 있는 것이죠.
해결책은 **비동기(Asynchronous)** 작업 큐입니다. "알았어, 나중에 해줄게"라고 응답하고, 뒤에서 조용히 처리하는 것입니다.

오늘부터 12주간, Python 생태계의 사실상 표준인 **Celery**와 가장 대중적인 인메모리 저장소 **Redis**를 이용하여, **수백만 건의 작업을 안정적으로 처리하는 분산 시스템**을 구축하는 방법을 배웁니다.

---

## 📅 12주 완성 커리큘럼 (Syllabus)

우리의 목표는 단순히 `delay()` 함수를 호출하는 것이 아닙니다. 작업이 실패했을 때 재시도하고, 여러 서버에 작업을 분산시키고, 순서를 보장하는 **엔지니어링**을 배웁니다.

### Part 1: Core Concepts (기초 원리)
*   **W1:** 오리엔테이션 & 비동기 작업의 필요성
*   **W2:** Celery 아키텍처 (Producer, Broker, Worker, Result Backend)
*   **W3:** Task 설계 패턴 (멱등성, 원자성) - *가장 중요!*
*   **W4:** Redis Broker 심화 (List vs Pub/Sub, Persistence)

### Part 2: Reliability & Orchestration (신뢰성과 조율)
*   **W5:** 에러 핸들링과 Retry 전략 (Exponential Backoff)
*   **W6:** Celery Canvas (Chain, Group, Chord) - 복잡한 워크플로우
*   **W7:** 주기적 작업 (Celery Beat & Cron)

### Part 3: Production Engineering (운영과 최적화)
*   **W8:** 모니터링 (Flower, Prometheus)
*   **W9:** 성능 최적화 (Prefetch Limit, Worker Concurrency)
*   **W10:** 라우팅 (Critical Queue vs Normal Queue)
*   **W11:** 테스트 전략 (Eager Mode, Mocking)
*   **W12:** Capstone: 대용량 이미지 처리 파이프라인 구축

---

## 🎓 1주차 강의: Why Async?

### 1. The Blocking Problem
전통적인 HTTP 요청-응답 주기는 다음과 같습니다.
1.  Client -> Server: "이메일 보내줘."
2.  Server: (이메일 서버 접속... 로그인... 전송... 대기...) -> 3초 소요.
3.  Server -> Client: "보냈어."

이 3초 동안 서버의 스레드 하나는 아무것도 못하고 기다립니다(Blocking). 사용자가 1000명이면? 서버는 다운됩니다.

### 2. The Task Queue Solution
중간에 **"메시지 브로커(Broker)"**를 둡니다.
1.  Client -> Server: "이메일 보내줘."
2.  Server -> Broker: (쪽지) "이메일 보내기 작업 추가." -> 0.001초 소요.
3.  Server -> Client: "접수했어(202 Accepted)."
4.  Worker -> Broker: "일거리 있어?" -> 쪽지 가져감 -> 이메일 전송.

사용자는 0.001초 만에 응답을 받습니다. 실제 전송은 Worker가 알아서 합니다.

---

## 🛠️ Lab: Environment Setup

실습을 위해 Docker로 Redis와 Celery 환경을 꾸밉니다.

### 1. `docker-compose.yml`
```yaml
version: '3.8'
services:
  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
```
`docker-compose up -d`로 Redis를 띄우세요.

### 2. Install Celery
```bash
pip install celery[redis]
```

### 3. Hello World
`tasks.py` 파일을 만듭니다.
```python
from celery import Celery
import time

app = Celery('tasks', broker='redis://localhost:6379/0', backend='redis://localhost:6379/1')

@app.task
def add(x, y):
    time.sleep(5) # 오래 걸리는 작업 시뮬레이션
    return x + y
```

### 4. Run Worker
터미널을 하나 더 열어서 워커를 실행합니다.
```bash
celery -A tasks worker --loglevel=info
```

### 5. Trigger Task
Python 쉘에서 작업을 던져봅니다.
```python
>>> from tasks import add
>>> result = add.delay(4, 4)
>>> result.ready()
False
>>> # 5초 후
>>> result.ready()
True
>>> result.get()
8
```

---

## 📝 1주차 과제: Synchronous vs Asynchronous Benchmark

**목표:** 동기 처리와 비동기 처리의 성능 차이를 체감하세요.

1.  **Sync:** `time.sleep(1)`을 포함한 함수를 `for` 문으로 10번 실행하는 코드 작성. 총 소요 시간 측정.
2.  **Async:** 위에서 만든 Celery `add` 함수(sleep 1초 포함)를 `for` 문으로 10번 `delay()` 호출. 작업 제출(Enqueuing)에 걸리는 시간 측정.
3.  두 결과의 차이를 비교하여 리포트로 제출하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 작업 큐 시스템에서, 생산자(Producer)와 소비자(Consumer) 사이에서 메시지를 중개해주는 컴포넌트를 무엇이라 하나요? (Broker)
2.  **Q2.** Celery에서 비동기 작업을 호출할 때 사용하는 메서드 이름은? (.delay() 또는 .apply_async())
3.  **Q3.** Redis를 브로커로 사용할 때, Celery는 어떤 자료구조를 사용하여 큐를 구현할까요? (List - LPUSH/BRPOP)

---

다음 주, Celery의 내부 구조를 뜯어보고, Redis가 어떻게 메시지를 전달하는지 깊이 파헤쳐 봅니다.

**Don't wait. Delegate.**
