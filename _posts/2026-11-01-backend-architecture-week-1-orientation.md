---
layout: post
title: "Backend Architecture Mastery: Week 1 - Orientation & The Async Revolution"
---

반갑습니다, 미래의 아키텍트 여러분.

저는 지난 30년간 금융권의 메인프레임부터 클라우드 네이티브 마이크로서비스까지, 수많은 시스템의 흥망성쇠를 지켜봐 왔습니다. 코드를 "작성"하는 것과 시스템을 "설계"하는 것은 완전히 다른 차원의 문제입니다. 코드는 컴파일러와 대화하지만, 아키텍처는 시간(Time) 그리고 규모(Scale)와 싸웁니다.

오늘부터 12주간, 우리는 단순히 "돌아가는 코드"를 넘어 **"무너지지 않는 시스템"**을 만드는 방법을 배울 것입니다. 특히 현대 백엔드의 핵심인 **비동기(Asynchronous)** 처리와 **분산 시스템(Distributed System)**의 원리를 파고듭니다.

여러분이 이 과정을 마칠 때쯤이면, 단순히 API를 만드는 개발자가 아니라, 트래픽이 폭주해도 우아하게 처리하는 **시스템 엔지니어**의 시야를 갖게 될 것입니다.

---

## 📅 12주 완성 커리큘럼 (Syllabus)

우리의 여정은 '동시성(Concurrency)'의 이해에서 시작해, 거대한 '분산 시스템'으로 확장됩니다.

### Part 1: Concurrency Fundamentals (동시성의 기초)
*   **W1:** 오리엔테이션 & Process vs Thread vs AsyncIO (GIL의 이해)
*   **W2:** Modern Python Web Servers (ASGI, Uvicorn, Gunicorn 아키텍처)
*   **W3:** FastAPI Internals (Dependency Injection & Pydantic 최적화)
*   **W4:** Background Task Patterns (In-process vs Out-of-process)

### Part 2: Distributed Message Passing (메시지 패싱과 큐)
*   **W5:** **RabbitMQ Deep Dive (AMQP, Exchanges, Reliability)**
*   **W6:** Reliability Engineering (Retries, Dead Letter Queues, Idempotency)
*   **W7:** Distributed Task Processing (Celery vs Dramatiq vs Custom Consumers)
*   **W8:** Event-Driven Architecture (Kafka vs RabbitMQ, Pub/Sub Patterns)

### Part 3: Scalability & Consistency (확장성과 일관성)
*   **W9:** Database Concurrency (ACID, Locks, MVCC in Async context)
*   **W10:** Caching Strategies at Scale (Redis, Thundering Herd, Cache Invalidation)
*   **W11:** Real-time Communication (WebSockets, SSE, Broadcasting)
*   **W12:** Capstone Project: High-Scale Job Management System 구축

---

## 🎓 1주차 강의: The Async Revolution

### 1. 커피숍 비유 (The Coffee Shop Analogy)

동기(Sync)와 비동기(Async)의 차이는 커피숍을 상상하면 가장 쉽습니다.

*   **Synchronous (Blocking):** 점원 한 명이 주문을 받고, 커피를 내리고, 손님에게 건네줄 때까지 다음 손님은 멍하니 기다립니다. 점원이 에스프레소 머신 앞에서 샷이 나오길 기다리는 동안 아무것도 하지 않죠. (CPU 낭비)
*   **Asynchronous (Non-Blocking):** 점원은 주문만 받고 진동벨을 줍니다. "다음 분!"을 외치죠. 커피를 내리는 건 머신(I/O Device)에게 맡기고, 점원(CPU)은 쉬지 않고 주문을 받습니다.

현대 웹 서비스는 I/O(DB 조회, API 호출)가 대부분입니다. 따라서 **점원(CPU)을 놀게 하지 않는 것**, 이것이 고성능의 핵심입니다.

### 2. Python의 GIL (Global Interpreter Lock)

"파이썬은 스레드를 써도 느리다"라는 말을 들어보셨을 겁니다. 바로 GIL 때문입니다.

*   **Process:** 독립된 메모리 공간. CPU 코어를 병렬로 사용 가능. (비쌈)
*   **Thread:** 메모리 공유. 문맥 전환(Context Switch) 비용 발생. GIL 때문에 한 번에 하나의 스레드만 Python 바이트코드를 실행.
*   **AsyncIO:** **단일 스레드** 내에서 루프(Event Loop)를 돌며 작업을 전환. 문맥 전환 비용이 거의 없음. (매우 가벼움)

I/O Bound 작업(네트워크, 디스크)이 주를 이루는 백엔드 서버에서는, **AsyncIO**가 멀티스레딩보다 압도적인 효율을 보여줍니다.

### 3. Event Loop의 마법

AsyncIO의 핵심은 **Event Loop**입니다.

```python
import asyncio

async def make_coffee():
    print("☕ 커피 내리기 시작 (I/O 요청)")
    await asyncio.sleep(2)  # 커피 머신이 일하는 동안 제어권을 반환!
    print("✅ 커피 완성")

async def take_order():
    print("🗣️ 주문 받기")
    await make_coffee()
    print("👋 손님 응대 끝")

async def main():
    # 두 명의 주문을 동시에 처리하는 것처럼 보임
    await asyncio.gather(take_order(), take_order())

asyncio.run(main())
```

`await` 키워드를 만나는 순간, 함수는 "나 잠깐 쉴게(IO 기다릴게), 다른 급한 일 먼저 해"라며 **제어권을 이벤트 루프에 반환**합니다. 이것이 수만 개의 요청을 동시에 처리할 수 있는 비결입니다.

---

## 🛠️ Lab: Blocking vs Non-Blocking

직접 코드를 돌려보며 체감해 봅시다.

### 준비물
*   Python 3.10 이상

### 실험 1: Blocking Code (Sync)
```python
import time

def sync_task(name):
    print(f"{name} 시작")
    time.sleep(2)  # CPU가 2초간 멈춤 (Blocking)
    print(f"{name} 끝")

start = time.time()
sync_task("A")
sync_task("B")
print(f"걸린 시간: {time.time() - start:.2f}초")
```
**결과:** 약 4초. (A가 끝날 때까지 B는 시작도 못 함)

### 실험 2: Non-Blocking Code (Async)
```python
import asyncio
import time

async def async_task(name):
    print(f"{name} 시작")
    await asyncio.sleep(2)  # 제어권 반환 (Non-Blocking)
    print(f"{name} 끝")

async def main():
    start = time.time()
    await asyncio.gather(async_task("A"), async_task("B"))
    print(f"걸린 시간: {time.time() - start:.2f}초")

asyncio.run(main())
```
**결과:** 약 2초. (A가 기다리는 동안 B가 실행됨)

이 2초의 차이가, 사용자가 100만 명일 때는 서버 비용 수억 원의 차이로 이어집니다.

---

## 📝 1주차 과제: Async Scraper

**목표:** 동기 방식과 비동기 방식으로 웹페이지 10개를 가져오는 코드를 각각 작성하고, 속도 차이를 측정하세요.

1.  대상 URL: `https://www.example.com` (혹은 응답이 1초 정도 걸리는 테스트 서버)
2.  **Sync 버전:** `requests` 라이브러리 사용. `for` 루프로 10번 호출.
3.  **Async 버전:** `aiohttp` 혹은 `httpx` 비동기 라이브러리 사용. `asyncio.gather`로 10번 동시 호출.
4.  두 코드의 실행 시간을 비교하여 리포트로 제출하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Python에서 멀티스레딩이 CPU Bound 작업에서 효율이 떨어지는 근본적인 원인은 무엇인가요? (G...)
2.  **Q2.** AsyncIO 함수 내에서 I/O 작업을 기다릴 때, 제어권을 이벤트 루프에 반환하기 위해 사용하는 키워드는? (a...)
3.  **Q3.** I/O Bound가 아닌, CPU 연산이 많은 작업(이미지 처리, 암호화 등)을 비동기 서버에서 처리해야 한다면 어떻게 해야 할까요? (힌트: ProcessPoolExecutor)

---

다음 주에는 이러한 비동기 기초를 바탕으로, 현대적인 Python 웹 서버의 표준인 **ASGI**와 **FastAPI**의 아키텍처를 뜯어보겠습니다.

**Stay Asynchronous.**
