---
layout: post
title: "Backend Architecture Mastery: Week 12 - Capstone Project"
---

축하합니다. 12주간의 긴 여정이 끝났습니다.
프로세스와 스레드에서 시작해서, 비동기 서버, 메시지 큐, 분산 처리, DB 락, 캐싱, 그리고 웹소켓까지.
이제 여러분은 **"시스템을 보는 눈"**이 달라졌을 겁니다.

마지막 주는 강의가 없습니다. 여러분이 직접 증명할 시간입니다.

---

## 🏆 Capstone Project: Enterprise Job Management System

지금까지 만든 조각코드들을 모아, 실제 상용 수준의 **"대용량 이미지 처리 파이프라인"**을 구축합니다.

### 1. System Requirements

**Frontend (Optional or Simple HTML):**
*   이미지 업로드 버튼
*   업로드된 작업의 상태(대기중 -> 처리중 -> 완료) 리스트
*   각 작업의 실시간 진행률 프로그레스 바

**Backend (FastAPI):**
*   `POST /jobs`: 이미지 업로드. 메타데이터 DB 저장. RabbitMQ로 메시지 발행.
*   `WS /ws/jobs/{user_id}`: 작업 상태 실시간 푸시.

**Worker (Celery or Custom):**
*   **Scale-out:** 워커 컨테이너를 최소 3개 이상 띄울 것.
*   **Task:** 이미지 흑백 변환 + 리사이징 (CPU 부하 시뮬레이션).
*   **Simulate:** 각 단계마다 `time.sleep`을 주고 Redis Pub/Sub으로 진행률 전송.

**Infrastructure (Docker Compose):**
*   API Server (FastAPI) x 2 (Load Balanced by Nginx)
*   Worker x 3
*   RabbitMQ (Message Broker)
*   Redis (Cache & Pub/Sub)
*   PostgreSQL (Main DB)

### 2. Constraints (제약사항)

1.  **Reliability:** 워커 하나를 강제로 죽여도(`docker stop`), 작업이 유실되지 않고 다른 워커가 이어서 처리해야 합니다. (Ack 메커니즘 확인)
2.  **Concurrency:** 동시에 100개의 이미지를 업로드해도 서버가 죽지 않아야 합니다. (Gunicorn/Uvicorn 설정 확인)
3.  **Idempotency:** 동일한 이미지를 실수로 두 번 업로드해도 처리는 한 번만 되어야 합니다. (Content Hash 등으로 중복 체크)

---

## 📝 Submission Guidelines

다음 내용을 포함한 GitHub Repository 링크와 리포트를 제출하세요.

1.  **Architecture Diagram:** `Draw.io` 등을 사용하여 시스템 전체 구조도 작성. 데이터 흐름(화살표) 표시 필수.
2.  **Docker Compose:** `docker-compose up` 한 방에 모든 시스템이 구동되어야 함.
3.  **Load Test Report:** `Locust` 등을 사용하여 동시 접속 100명일 때의 평균 응답 시간과 처리량(RPS) 측정 결과.
4.  **Troubleshooting Log:** 개발 중 겪었던 동시성 문제나 버그, 그리고 어떻게 해결했는지에 대한 기록.

---

## 🎓 Closing Remarks

이 프로젝트를 완성한다면, 여러분은 어디 가서 **"백엔드 아키텍처 좀 안다"**고 말하셔도 좋습니다.
단순히 API를 만드는 것을 넘어, **트래픽을 감당하고, 실패를 견뎌내며, 유연하게 확장하는 시스템**을 설계할 수 있게 되었으니까요.

30년 전 메인프레임을 만지던 시절부터 지금까지 변하지 않는 진리는 하나입니다.
**"완벽한 시스템은 없다. 끊임없이 개선되는 시스템만 있을 뿐이다."**

여러분의 코드가 세상을 지탱하는 단단한 기둥이 되길 바랍니다.
수고하셨습니다.

**Build Strong, Scale High.**
