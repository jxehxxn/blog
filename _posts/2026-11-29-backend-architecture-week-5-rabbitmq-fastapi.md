---
layout: post
title: "Backend Architecture Mastery: Week 5 - RabbitMQ Deep Dive & FastAPI Job Manager"
---

지난주까지 우리는 단일 서버 내에서의 비동기 처리(`asyncio`)에 대해 배웠습니다. 하지만 서버 한 대의 메모리 안에서 모든 것을 처리하려는 순간, 시스템은 위험해집니다. 서버가 재시작되면 진행 중이던 작업은 증발하고, 트래픽이 폭주하면 서버는 `Out of Memory`로 죽어버리죠.

오늘은 서버의 경계를 넘어, **RabbitMQ**라는 강력한 메시지 브로커를 이용해 **작업을 안전하게 위임하고(Offloading)**, **시스템을 분리(Decoupling)**하는 방법을 배웁니다.

특히, 실무에서 가장 많이 쓰이는 패턴인 **"FastAPI 비동기 Job Manager"**를 처음부터 끝까지 구현해 보겠습니다.

---

## 🐇 Why RabbitMQ?

Redis Pub/Sub도 있고, Kafka도 있는데 왜 RabbitMQ일까요?

1.  **Smart Broker, Dumb Consumer:** RabbitMQ는 라우팅 규칙(Exchange)이 매우 강력합니다. 메시지를 어디로 보낼지 브로커가 결정해주므로, 컨슈머는 단순히 "일만 하면" 됩니다.
2.  **Reliability:** 메시지가 처리되었는지 확인하는 `Ack`(Acknowledgement) 메커니즘이 강력합니다. 컨슈머가 일을 하다가 죽으면? RabbitMQ는 그 메시지를 다시 큐에 넣어서 다른 컨슈머에게 줍니다. (메시지 유실 방지)
3.  **Standard Protocol:** AMQP(Advanced Message Queuing Protocol)라는 표준을 따르므로, 언어에 구애받지 않습니다. Python에서 보내고 Java에서 받아도 됩니다.

---

## 🏗️ Architecture: The Job Manager Pattern

우리가 만들 시스템의 구조입니다.

1.  **Client:** `POST /jobs`로 무거운 작업(예: 이미지 변환, 리포트 생성)을 요청합니다.
2.  **FastAPI (Producer):** 요청을 받자마자 **"알았다(Accepted 202)"**고 응답하고, RabbitMQ에 "이 작업 좀 해줘"라고 메시지 쪽지를 넣습니다. 그리고 `job_id`를 즉시 반환합니다.
3.  **RabbitMQ (Broker):** 메시지를 안전하게 보관합니다.
4.  **Worker (Consumer):** 별도의 프로세스로 돌고 있는 워커가 메시지를 하나씩 꺼내서 실제 무거운 작업을 수행합니다.
5.  **Redis (Result Backend):** 워커가 작업을 마치면 결과를 Redis에 저장합니다.
6.  **Client:** `GET /jobs/{job_id}`로 작업 상태를 확인합니다.

이 구조의 핵심은 **"사용자를 기다리게 하지 않는다"**입니다.

---

## 🛠️ Implementation

자, 코드로 들어갑시다. `aio_pika` 라이브러리를 사용하여 완전한 비동기로 구현합니다.

### 1. Environment Setup (`docker-compose.yml`)

RabbitMQ와 Redis를 띄웁니다.

```yaml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"   # AMQP protocol
      - "15672:15672" # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: user
      RABBITMQ_DEFAULT_PASS: password

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
```

### 2. Dependency Injection (`deps.py`)

RabbitMQ 연결은 비용이 비쌉니다. 매 요청마다 연결하지 않고, 풀(Pool)을 쓰거나 싱글톤으로 관리해야 합니다. 여기서는 FastAPI의 수명 주기(Lifespan)를 활용합니다.

```python
# deps.py
import aio_pika
import os

RABBITMQ_URL = "amqp://user:password@localhost:5672/"

async def get_rabbitmq_channel():
    connection = await aio_pika.connect_robust(RABBITMQ_URL)
    async with connection:
        channel = await connection.channel()
        yield channel
```

### 3. FastAPI Producer (`main.py`)

```python
# main.py
from fastapi import FastAPI, Depends
from pydantic import BaseModel
import uuid
import json
import aio_pika
from deps import get_rabbitmq_channel

app = FastAPI()

class JobRequest(BaseModel):
    task_name: str
    payload: dict

@app.post("/jobs", status_code=202)
async def create_job(
    job: JobRequest,
    channel: aio_pika.abc.AbstractChannel = Depends(get_rabbitmq_channel)
):
    job_id = str(uuid.uuid4())
    message_body = {
        "job_id": job_id,
        "task": job.task_name,
        "data": job.payload
    }
    
    # Default Exchange를 사용하여 'task_queue'라는 큐로 바로 보냅니다.
    await channel.default_exchange.publish(
        aio_pika.Message(
            body=json.dumps(message_body).encode(),
            delivery_mode=aio_pika.DeliveryMode.PERSISTENT # 메시지 영속성 보장
        ),
        routing_key="task_queue"
    )
    
    return {"job_id": job_id, "status": "queued"}
```

### 4. The Worker (`worker.py`)

이 스크립트는 백그라운드에서 계속 실행되며 큐를 감시합니다.

```python
# worker.py
import asyncio
import aio_pika
import json
import time

RABBITMQ_URL = "amqp://user:password@localhost:5672/"

async def process_task(message: aio_pika.abc.AbstractIncomingMessage):
    async with message.process(): # 완료 후 자동으로 Ack 전송
        body = json.loads(message.body)
        print(f" [x] Received Job {body['job_id']}: {body['task']}")
        
        # 무거운 작업을 시뮬레이션 (여기서 AI 추론, 이미지 처리 등을 수행)
        await asyncio.sleep(5) 
        
        print(f" [v] Job {body['job_id']} Done!")
        # 실제로는 여기서 결과를 DB나 Redis에 저장합니다.

async def main():
    connection = await aio_pika.connect_robust(RABBITMQ_URL)
    channel = await connection.channel()
    
    # 큐가 없으면 만듭니다. (durable=True로 설정하여 RabbitMQ 재시작 후에도 유지)
    queue = await channel.declare_queue("task_queue", durable=True)
    
    # Fair Dispatch: 한 번에 하나씩만 처리하겠다는 설정 (QoS)
    await channel.set_qos(prefetch_count=1)
    
    print(' [*] Waiting for messages. To exit press CTRL+C')
    await queue.consume(process_task)
    
    await asyncio.Future() # 무한 대기

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 🔍 Deep Dive: Message Reliability

프로덕션 레벨에서는 무엇이 중요할까요? 바로 **"메시지 유실 방지"**입니다. 위 코드에는 3가지 안전장치가 숨어있습니다.

1.  **Durable Queue:** `declare_queue(..., durable=True)`
    *   RabbitMQ가 재시작되어도 큐 자체가 사라지지 않습니다.
2.  **Persistent Message:** `DeliveryMode.PERSISTENT`
    *   메시지를 디스크에 저장합니다. 브로커가 셧다운되어도 메시지는 살아남습니다.
3.  **Manual Ack:** `async with message.process():`
    *   이 컨텍스트 매니저는 작업이 에러 없이 끝나야만 `Ack`를 보냅니다.
    *   만약 작업 도중 워커가 죽으면(Exception or Kill), `Ack`가 오지 않았으므로 RabbitMQ는 이 메시지를 다시 큐에 넣습니다. 다른 워커가 가져갈 수 있게요.

---

## 📝 5주차 과제: Retry Mechanism 구현

현실 세계는 지저분합니다. 외부 API가 500 에러를 뱉기도 하고, DB가 잠시 끊기기도 하죠.

**목표:** 워커가 작업 처리 중 실패했을 때, 바로 버리는 게 아니라 **3번까지 재시도(Retry)**하도록 `worker.py`를 수정하세요.

**힌트:**
1.  `try-except` 블록을 사용하세요.
2.  실패 시 `nack(requeue=False)`를 하고, 메시지 헤더에 `retry_count`를 추가하여 다시 `publish` 하세요.
3.  Dead Letter Exchange(DLX) 개념을 찾아보세요. 3번 다 실패하면 '무덤' 큐(DLQ)로 보내야 합니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** RabbitMQ에서 메시지의 라우팅 규칙(어느 큐로 보낼지)을 결정하는 논리적 개념은 무엇인가요? (E...)
2.  **Q2.** 컨슈머가 메시지를 잘 받았고 처리했다고 브로커에게 알리는 신호를 무엇이라고 하나요? (A...)
3.  **Q3.** 메시지가 큐에 쌓이는 속도가 소비하는 속도보다 빠를 때, 컨슈머가 한 번에 가져갈 메시지 개수를 제한하는 설정은? (QoS P...)

---

다음 주에는 **Dead Letter Queue**와 **Celery** 같은 고수준 프레임워크가 이 모든 것을 어떻게 추상화했는지 비교해 보겠습니다.

**Keep Queuing.**
