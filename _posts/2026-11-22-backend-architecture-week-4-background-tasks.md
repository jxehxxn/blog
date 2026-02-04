---
layout: post
title: "Backend Architecture Mastery: Week 4 - Background Task Patterns"
---

API 응답 속도는 사용자 경험(UX)의 핵심입니다. 사용자가 "회원가입" 버튼을 눌렀는데, 환영 이메일을 보내느라 3초 동안 로딩 바가 돈다면? 사용자는 떠납니다.

이메일 발송, 이미지 리사이징, 로그 집계... 이런 건 사용자를 기다리게 할 필요가 없습니다. "응답 먼저 주고, 일은 나중에" 하면 되죠. 오늘은 이것을 가능하게 하는 **Background Tasks** 패턴을 배웁니다.

---

## 1. FastAPI `BackgroundTasks`: Simple & Built-in

FastAPI는 아주 간단한 백그라운드 작업 도구를 내장하고 있습니다.

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def send_email(email: str, message: str):
    # 시간이 걸리는 작업 (동기 함수여도 됨)
    import time
    time.sleep(2) 
    print(f"📧 Sent email to {email}")

@app.post("/send-notification/{email}")
async def send_notification(email: str, background_tasks: BackgroundTasks):
    background_tasks.add_task(send_email, email, message="Welcome!")
    return {"message": "Notification sent in the background"}
```

### 어떻게 동작하나?
응답(`return`)이 반환된 **직후**, FastAPI가 `background_tasks`에 등록된 함수들을 실행합니다. 이 함수들은 여전히 **FastAPI 서버 프로세스 내부**에서 실행됩니다.

### 언제 쓰는가?
*   매우 가벼운 작업 (로그 남기기, 간단한 알림)
*   실패해도 치명적이지 않은 작업
*   외부 인프라(Redis, RabbitMQ)를 구축하기 귀찮을 때

---

## 2. The Danger of In-Process Tasks

"와, 너무 쉬운데요? 그냥 이걸로 다 하면 안 되나요?"

절대 안 됩니다. `BackgroundTasks`는 치명적인 한계가 있습니다.

1.  **Memory Leak:** 작업이 계속 쌓이면 서버 메모리가 터집니다.
2.  **Server Restart = Data Loss:** 서버를 배포하거나 재시작하면, 큐에 있던 작업들은 싹 사라집니다. (휘발성)
3.  **CPU Blocking:** 파이썬은 GIL이 있다고 했죠? 백그라운드 작업이 CPU를 많이 쓰면, 메인 API 응답도 덩달아 느려집니다.

그래서 실무에서는 **"Out-of-Process"** 방식, 즉 별도의 작업 큐(Task Queue) 시스템이 필수적입니다.

---

## 3. From In-Process to Out-of-Process

서버 아키텍처를 진화시켜 봅시다.

*   **Level 1 (Synchronous):** 요청 -> 이메일 발송(3초) -> 응답. (최악)
*   **Level 2 (Async/BackgroundTasks):** 요청 -> 작업 등록 -> 응답 -> (서버 내에서) 이메일 발송. (소규모 적합)
*   **Level 3 (Task Queue):** 요청 -> 메시지 큐에 "이메일 보내" 쪽지 투척 -> 응답. 별도의 Worker 서버가 쪽지를 주워가서 처리. (대규모 표준)

다음 주(Week 5)에 다룰 RabbitMQ가 바로 Level 3의 핵심입니다. 오늘은 그 징검다리로서, 비동기 작업의 개념을 확실히 잡고 갑시다.

---

## 🛠️ Lab: Don't Block the Event Loop

`BackgroundTasks`를 쓸 때 가장 많이 하는 실수를 재현해 봅니다.

### 잘못된 예시 (Event Loop 차단)
```python
async def cpu_intensive_task():
    # 엄청난 계산 작업 (CPU 100%)
    total = 0
    for i in range(100000000):
        total += i

@app.post("/bad")
async def bad_handler(background_tasks: BackgroundTasks):
    background_tasks.add_task(cpu_intensive_task)
    return {"message": "Server will freeze soon"}
```
비록 응답은 빨리 나가지만, 직후에 실행되는 `cpu_intensive_task`가 이벤트 루프를 독점해버립니다. 그동안 들어오는 다른 사람들의 요청(`GET /`)은 타임아웃이 날 겁니다.

### 올바른 예시 (Thread/Process Pool)
CPU 작업은 별도 스레드나 프로세스로 던져야 합니다.

```python
from fastapi.concurrency import run_in_threadpool

async def good_cpu_task():
    await run_in_threadpool(cpu_function)
```

---

## 📝 4주차 과제: Image Resizer (In-Process)

**목표:** 사용자가 이미지를 업로드하면, 원본은 저장하고 썸네일 생성은 백그라운드에서 수행하는 API를 만드세요.

1.  `Pillow` 라이브러리 사용.
2.  `POST /upload`: 이미지 파일 업로드.
3.  응답으로 `filename` 즉시 반환.
4.  백그라운드에서: 이미지를 읽어 가로 100px로 리사이징하고 `thumb_{filename}`으로 저장.
5.  `time.sleep(5)`를 추가하여 오래 걸리는 작업임을 시뮬레이션할 것.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** FastAPI의 `BackgroundTasks`는 서버가 강제 종료되면 실행 중이던 작업이 어떻게 되나요? (사라짐)
2.  **Q2.** 비동기 함수(`async def`) 내에서 `time.sleep(10)`을 쓰면 발생하는 문제는? (전체 서버 멈춤. `asyncio.sleep`을 써야 함)
3.  **Q3.** CPU 연산이 많은 작업(AI 추론 등)을 FastAPI 내부 `BackgroundTasks`로 돌리는 것이 권장되지 않는 이유는? (GIL로 인한 성능 저하)

---

자, 이제 서버 내부에서 할 수 있는 건 다 해봤습니다.
다음 주, 드디어 서버 밖으로 나갑니다. **RabbitMQ**와 함께 진정한 분산 시스템의 세계로 떠납니다.

**Think Outside the Process.**
