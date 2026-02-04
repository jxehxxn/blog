---
layout: post
title: "Backend Architecture Mastery: Week 2 - Modern Python Web Servers"
---

지난주, 우리는 Python의 비동기 혁명(AsyncIO)에 대해 이야기했습니다. 하지만 `async def`로 짠 코드를 실제로 인터넷 세상에 노출하려면 "서버"가 필요합니다.

"그냥 `python main.py` 하면 되는 거 아닌가요?"

개발 단계에선 그렇습니다. 하지만 프로덕션 레벨에서는 이야기가 다릅니다. 오늘은 Python 웹 서버의 진화 과정인 **WSGI**와 **ASGI**, 그리고 **Uvicorn**과 **Gunicorn**의 완벽한 공생 관계를 파헤쳐 봅니다.

---

## 1. WSGI vs ASGI: The Protocol War

### WSGI (Web Server Gateway Interface)
2000년대 초반, Python 웹 프레임워크(Django, Flask)를 위해 만들어진 표준입니다.
*   **특징:** 동기(Synchronous) 방식. 요청 하나당 스레드 하나를 점유합니다.
*   **한계:** WebSocket, Long Polling 같은 비동기 통신을 처리하기엔 구조적으로 부적합합니다.

### ASGI (Asynchronous Server Gateway Interface)
AsyncIO의 등장과 함께 태어난 새로운 표준입니다.
*   **특징:** 비동기(Asynchronous) 방식. 단일 스레드/프로세스 내에서 수천 개의 연결을 동시에 유지할 수 있습니다.
*   **대상:** FastAPI, Django Channels.

**비유:**
*   **WSGI:** 전화 상담원. 한 번에 한 명의 고객과 통화가 끝날 때까지 다른 전화를 못 받습니다.
*   **ASGI:** 카카오톡 상담원. 여러 채팅방을 오가며 동시에 수십 명과 대화합니다.

---

## 2. Uvicorn & Gunicorn: The Dynamic Duo

FastAPI를 배포할 때 공식 문서에는 이렇게 적혀 있습니다.
`gunicorn -k uvicorn.workers.UvicornWorker main:app`

도대체 왜 서버를 두 개나 쓸까요?

### Uvicorn: The Lightning ⚡
*   **역할:** **ASGI 서버.** 비동기 요청을 실제로 처리하고, Event Loop를 돌리는 엔진입니다. `uvloop` (C로 작성된 엄청 빠른 루프) 위에서 돌아갑니다.
*   **단점:** 프로세스 관리 능력이 부족합니다. 혼자 죽으면 되살아나지 못합니다.

### Gunicorn: The Manager 👔
*   **역할:** **Process Manager.** UNIX 시스템에서 프로세스를 관리하는 데 특화되어 있습니다.
*   **기능:** 워커 프로세스 갯수 관리, 죽은 프로세스 자동 재시작, 로드 밸런싱.
*   **단점:** 비동기 처리는 못 합니다.

### 결론: The Perfect Marriage
**Gunicorn이 사장님(Manager)**이 되어 여러 명의 **Uvicorn 직원(Worker)**을 고용하는 형태입니다. 사장님은 직원들이 딴짓 안 하는지, 죽진 않았는지 감시하고, 직원들은 실무(비동기 처리)를 빡세게 합니다.

---

## 🛠️ Lab: Stress Testing Uvicorn vs Gunicorn

서버 구성에 따른 성능 차이를 눈으로 확인해 봅시다.

### 준비물
*   FastAPI, Uvicorn, Gunicorn
*   `wrk` 또는 `ab` (Apache Benchmark) 같은 부하 테스트 도구

### 시나리오
1초가 걸리는 비동기 API를 만들고, 동시 접속자 100명이 요청을 보낼 때를 가정합니다.

```python
# main.py
from fastapi import FastAPI
import asyncio

app = FastAPI()

@app.get("/")
async def root():
    await asyncio.sleep(1) # I/O Bound 작업 시뮬레이션
    return {"message": "Hello"}
```

### 실험 1: Uvicorn 단독 실행 (Process 1개)
```bash
uvicorn main:app --port 8000
# 다른 터미널에서
wrk -t12 -c100 -d10s http://127.0.0.1:8000
```

### 실험 2: Gunicorn + Uvicorn (Process 4개)
```bash
gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app
# 다른 터미널에서
wrk -t12 -c100 -d10s http://127.0.0.1:8000
```

**예상 결과:**
CPU 코어가 여러 개인 머신이라면, 실험 2가 실험 1보다 **약 3~4배 더 높은 처리량(RPS)**을 보여줄 것입니다. 이것이 **멀티 프로세스 + 비동기**의 힘입니다.

---

## 📝 2주차 과제: Custom Config Deployment

**목표:** 여러분의 FastAPI 애플리케이션을 위한 프로덕션급 `gunicorn.conf.py` 설정 파일을 작성하세요.

**필수 포함 설정:**
1.  **Workers:** CPU 코어 수에 기반한 적절한 워커 수 계산 (`(CPU Core * 2) + 1` 공식 활용)
2.  **Bind:** Unix Socket 사용 (Nginx와 연동 시 성능 향상) 혹은 `0.0.0.0:8000`
3.  **Logs:** Access Log와 Error Log 파일 경로 지정
4.  **Timeout:** 긴 요청이 들어올 경우를 대비한 적절한 타임아웃 설정

---

## ✅ Self-Assessment Quiz

1.  **Q1.** FastAPI는 WSGI 기반인가요, ASGI 기반인가요? (A...)
2.  **Q2.** Uvicorn이 사용하는 고성능 Event Loop의 이름은 무엇인가요? (u...)
3.  **Q3.** Gunicorn 없이 Uvicorn만으로 프로덕션에 배포했을 때 가장 큰 위험 요소는 무엇인가요? (단일 프로세스의 취약점)

---

다음 주에는 FastAPI 내부로 들어가, **Pydantic**이 어떻게 데이터를 검증하고 직렬화하는지, 그리고 **Dependency Injection** 시스템이 얼마나 우아한지 살펴보겠습니다.

**Don't block the loop.**
