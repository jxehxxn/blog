---
layout: post
title: "Backend Architecture Mastery: Week 11 - Real-time Communication (WebSockets)"
---

HTTP는 야속합니다. 클라이언트가 물어봐야만 대답해줍니다.
"새 메시지 왔어?" "아니."
"새 메시지 왔어?" "아니."
"새 메시지 왔어?" "응." (Polling)

이건 비효율적입니다. 서버가 먼저 "야! 메시지 왔어!"라고 말해주면 얼마나 좋을까요?
그것이 바로 **WebSocket**입니다.

---

## 1. Polling vs Long Polling vs WebSockets

1.  **Polling:** 1초마다 물어봄. (서버 부하 큼, 실시간성 떨어짐)
2.  **Long Polling:** 물어보면, 대답할 게 생길 때까지 안 끊고 기다림. (HTTP 연결을 오래 잡고 있음. 조금 나음)
3.  **WebSocket:** 전화 연결(TCP)을 한번 하면 끊지 않고 양방향으로 막 떠듦. (Real-time, 오버헤드 적음)

---

## 2. FastAPI WebSockets

FastAPI는 WebSocket을 아주 우아하게 지원합니다.

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

class ConnectionManager:
    def __init__(self):
        self.active_connections: list[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: int):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"Client #{client_id} says: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast(f"Client #{client_id} left")
```

이 코드 하나면 단체 채팅방이 완성됩니다.

---

## 3. Scale-out Problem (Pub/Sub Redis)

문제는 서버가 여러 대일 때입니다.
*   User A는 Server 1에 붙어있고,
*   User B는 Server 2에 붙어있습니다.
*   A가 메시지를 보내면, Server 1의 `manager.active_connections`에는 B가 없습니다. B는 메시지를 못 받습니다.

해결책은? **Redis Pub/Sub**입니다.
1.  Server 1이 메시지를 받으면 -> Redis 채널에 `PUBLISH`.
2.  모든 서버(Server 1, 2, 3...)는 Redis 채널을 `SUBSCRIBE` 하고 있음.
3.  Redis에서 메시지가 오면 -> 자기한테 붙어있는 WebSocket 클라이언트들에게 쏴줌.

---

## 🛠️ Lab: Real-time Job Progress

우리가 5주차에 만든 Job Manager에 실시간 진행률 표시 기능을 붙여봅시다.

1.  Worker가 작업을 처리하면서 중간중간 Redis Pub/Sub으로 진행률(`{"progress": 50}`)을 쏩니다.
2.  FastAPI는 WebSocket으로 클라이언트와 연결되어 있습니다.
3.  FastAPI가 Redis 메시지를 받아서 WebSocket으로 전달합니다.
4.  프론트엔드에서는 프로그레스 바가 실시간으로 쭉쭉 찹니다.

---

## 📝 11주차 과제: Stock Ticker

**목표:** 가상화폐 거래소의 시세판을 구현하세요.

1.  백그라운드에서 1초마다 비트코인 가격을 랜덤하게 생성하여 Redis Pub/Sub으로 쏘는 `Ticker Producer`를 만듭니다.
2.  FastAPI에서 WebSocket을 열고, 클라이언트가 접속하면 현재 가격을 실시간으로(1초마다) 전송합니다.
3.  `wscat` 혹은 간단한 HTML/JS 클라이언트로 접속하여 가격이 변하는 것을 캡처하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** HTTP 프로토콜을 WebSocket 프로토콜로 업그레이드하기 위해 사용하는 헤더는? (Upgrade: websocket, Connection: Upgrade)
2.  **Q2.** WebSocket은 TCP 위에서 동작하나요 UDP 위에서 동작하나요? (TCP)
3.  **Q3.** 서버 -> 클라이언트 단방향 통신만 필요할 때, WebSocket보다 가볍고 재접속 처리가 쉬운 HTTP 기반 기술은? (SSE - Server-Sent Events)

---

다음 주, 대망의 **마지막 주차**입니다.
지금까지 배운 모든 기술(FastAPI, Celery, RabbitMQ, Redis, WebSocket, Docker)을 총동원하여 하나의 거대한 시스템을 만듭니다.

**Stay Connected.**
