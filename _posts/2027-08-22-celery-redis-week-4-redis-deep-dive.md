---
layout: post
title: "Distributed Task Processing with Celery & Redis: Week 4 - Redis as a Broker & Backend Deep Dive"
---

Week 1 보충 강의에서 Redis와 RabbitMQ를 비교했습니다.
이번 주에는 Celery의 관점에서 **Redis를 극한으로 활용하는 방법**을 배웁니다.

"Redis는 그냥 꽂으면 되는 거 아닌가요?"
천만에요. `Visibility Timeout` 설정을 잘못해서 무한 재시도 루프에 빠지거나, 메모리 폭발로 서버가 죽는 경우가 허다합니다.

---

## 1. Visibility Timeout: The Invisible Trap

Broker는 Worker에게 작업을 줄 때, 큐에서 바로 삭제하지 않습니다.
대신 **"예약 중(Reserved)"** 상태로 만듭니다. 다른 Worker는 못 가져가게 숨기는 거죠.

*   **Visibility Timeout:** "이 시간 안에 ACK(완료 신호)가 안 오면, 워커가 죽은 걸로 간주하고 다시 큐에 보이게 할게."

**문제점:** 작업이 실제로 1시간 걸리는데, Timeout이 30분이면?
1.  워커 A가 30분째 일하는 중.
2.  Broker: "30분 지났네? A 죽었나 봐." -> 작업 부활.
3.  워커 B가 같은 작업을 가져감.
4.  **중복 실행 발생.**

**해결:** `broker_transport_options`의 `visibility_timeout`을 예상되는 최대 작업 시간보다 길게 설정해야 합니다.

```python
app.conf.broker_transport_options = {
    'visibility_timeout': 3600, # 1시간
}
```

---

## 2. Redis Persistence for Reliability

Redis는 메모리 기반입니다. 전원 나가면 작업 다 날아갑니다.
이를 방지하기 위해 두 가지 영속성 옵션을 씁니다.

### RDB (Snapshot)
*   일정 간격(예: 5분)마다 메모리 전체를 디스크에 덤프.
*   빠르지만, 덤프 사이의 데이터는 유실 가능.

### AOF (Append Only File)
*   모든 쓰기 연산을 로그 파일에 기록.
*   데이터 유실 거의 없음. 파일 크기가 커지고 느림.

**Queue를 위한 Best Practice:**
브로커용 Redis라면 **AOF**를 켜거나, 아예 날아가도 되는 작업만 처리하세요. 절대 잃어버리면 안 되는 작업은 RabbitMQ를 쓰세요.

---

## 3. Redis as Result Backend: TTL

결과 백엔드로 Redis를 쓸 때 주의할 점은 **메모리 누수(Memory Leak)**입니다.
Task가 완료될 때마다 결과가 Redis에 저장되는데, 아무도 안 찾아가면 영원히 남습니다.

반드시 **만료 시간(result_expires)**을 설정해야 합니다.

```python
app.conf.result_expires = 60 * 60 * 24 # 24시간 후 자동 삭제
```

---

## 4. Priority Queue (Redis Priority Steps)

Redis는 기본적으로 우선순위 큐를 지원하지 않습니다. (List는 FIFO)
Celery는 이를 흉내 내기 위해 여러 개의 List를 만들고 순서대로 조회하는 방식을 씁니다.

*   `celery` (priority 0)
*   `celery\x06\x163` (priority 3)
*   `celery\x06\x166` (priority 6)
*   `celery\x06\x169` (priority 9)

워커는 높은 숫자 큐부터 확인합니다.

---

## 🛠️ Lab: Monitoring Redis with CLI

작업이 쌓였을 때 Redis 내부를 들여다봅니다.

1.  긴 작업(`sleep 60`)을 10개 던집니다.
2.  `redis-cli` 접속.
3.  `LLEN celery`: 대기 중인 작업 수 확인.
4.  `LRANGE celery 0 -1`: 작업 내용(JSON) 확인.
5.  `KEYS *`: 어떤 키들이 생성되었는지 확인 (`_kombu.binding.celery` 등).

---

## 📝 4주차 과제: Configure Visibility Timeout

**목표:** 긴 작업을 위해 설정을 튜닝하세요.

1.  5분(`time.sleep(300)`) 걸리는 Task를 만듭니다.
2.  Celery 설정에서 `visibility_timeout`을 **10초**로 설정합니다. (일부러 짧게)
3.  Task를 실행하고 로그를 관찰하세요.
4.  10초마다 작업이 재실행(중복 실행)되는 현상을 캡처하고, 왜 그런지 이유를 설명하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Worker가 작업을 수행 중인데 Broker가 이를 "실패했다"고 오판하여 재할당하는 것을 막기 위해 조절해야 하는 시간 설정은? (Visibility Timeout)
2.  **Q2.** Redis의 Persistence 방식 중, 모든 쓰기 명령을 로그로 남겨 데이터 유실을 최소화하는 방식은? (AOF - Append Only File)
3.  **Q3.** Celery Result Backend에 결과가 무한히 쌓여 Redis 메모리가 가득 차는 것을 막기 위한 설정은? (result_expires / Result Expiration)

---

다음 주, 에러는 피할 수 없습니다. 현명하게 대처할 뿐입니다.
**Retry Strategies**와 **Dead Letter Queue**에 대해 배웁니다.

**Timeout is ticking.**
