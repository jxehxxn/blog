---
layout: post
title: "Distributed Task Processing with Celery & Redis: Week 12 - Capstone: Scalable Image Processing Pipeline"
---

12주간의 여정이 끝났습니다.
단순한 `delay()` 호출에서 시작해, 이제는 멱등성, 라우팅, 모니터링까지 고려하는 분산 시스템 엔지니어가 되었습니다.

마지막 과제는 이 모든 것을 종합한 **프로덕션 수준의 파이프라인** 구축입니다.

---

## 🏆 Capstone Project

**주제:** 인스타그램 스타일의 "대용량 이미지 필터링 & 업로드 서비스"

### 1. Requirements

1.  **API:** 사용자가 이미지 URL 목록(100개 이상)을 업로드.
2.  **Pipeline:**
    *   다운로드 -> 리사이징(썸네일) -> 흑백 변환 -> S3 업로드.
    *   이 과정은 병렬로 처리되어야 함 (`Chunk` or `Group`).
    *   모든 이미지가 처리되면 사용자에게 "완료" 이메일 발송 (`Chord`).
3.  **Reliability:**
    *   외부 이미지 다운로드 실패 시 3회 재시도 (Exponential Backoff).
    *   Redis가 재시작되어도 작업이 유실되지 않아야 함 (AOF).
4.  **Routing:**
    *   일반 사용자는 `default` 큐.
    *   Premium 사용자는 `high_priority` 큐.
5.  **Monitoring:**
    *   Flower로 실시간 모니터링 가능해야 함.

### 2. Deliverables

1.  **Source Code:** Django/Flask + Celery 프로젝트.
2.  **Architecture Diagram:** Producer, Redis, Worker Group, S3, Email 간의 흐름도.
3.  **Load Test Report:** 이미지 1000개를 동시에 요청했을 때 큐가 어떻게 쌓이고 빠지는지 Flower 스크린샷과 함께 설명.

---

## 3. Production Checklist

배포 전 마지막 점검 리스트입니다.

*   [ ] **Broker URL:** `redis://`에 비밀번호가 걸려있는가?
*   [ ] **Security:** Pickle을 끄고 JSON만 사용하는가?
*   [ ] **Resource:** Worker에 Memory Limit(`--max-memory-per-child`)이 걸려있는가?
*   [ ] **Monitoring:** Prometheus/Grafana 알람이 설정되어 있는가?

---

## 🎓 Closing Remarks

분산 시스템은 어렵습니다. 디버깅도 힘들고, 고려할 변수도 많습니다.
하지만 그만큼 강력합니다. 서버 한 대로는 불가능한 일을 가능하게 만듭니다.

여러분이 배운 **"작업을 쪼개고, 줄 세우고, 나눠서 처리하는 기술"**은 Celery뿐만 아니라 Kafka, SQS 등 모든 비동기 시스템에 통용되는 원리입니다.

이제 여러분은 어떤 트래픽이 몰려와도 "잠시만요, 큐에 넣고 처리해드릴게요"라고 여유 있게 말할 수 있는 엔지니어입니다.
수고하셨습니다.

**Queue empty. Worker idle.**
