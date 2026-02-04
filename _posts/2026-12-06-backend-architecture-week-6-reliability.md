---
layout: post
title: "Backend Architecture Mastery: Week 6 - Reliability Engineering"
---

지난주(Week 5)에 RabbitMQ로 작업을 분산시키는 법을 배웠습니다. 하지만 분산 시스템에는 "8가지 오류(Fallacies of Distributed Computing)"가 있습니다. 그중 첫 번째가 바로 **"네트워크는 신뢰할 수 있다"**입니다. 거짓말이죠. 네트워크는 반드시 실패합니다.

오늘은 시스템이 실패해도 **결국에는 성공하게 만드는(Eventual Consistency)** 신뢰성 공학(Reliability Engineering)을 배웁니다.

---

## 1. Retry Strategies: 포기하지 않는 끈기

API 호출이 실패했다고 바로 에러를 띄우는 건 하수입니다. 잠깐 네트워크가 끊긴 걸 수도 있잖아요?

### Exponential Backoff (지수 백오프)
무작정 1초마다 재시도하면 안 됩니다. 서버가 힘들어할 때 더 때리는 꼴이니까요.
*   1회차: 1초 후 재시도
*   2회차: 2초 후 재시도
*   3회차: 4초 후 재시도
*   4회차: 8초 후 재시도...

Python의 `tenacity` 라이브러리를 쓰면 아주 우아하게 구현할 수 있습니다.

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(5), wait=wait_exponential(multiplier=1, min=2, max=10))
def unstable_api_call():
    print("Trying...")
    raise Exception("Network Fail")
```

---

## 2. Dead Letter Queue (DLQ): 실패의 무덤

재시도를 10번 했는데도 안 되면? 계속 잡고 있으면 안 됩니다. 큐만 막히죠.
이때 메시지를 **"죽은 편지 보관함(Dead Letter Queue)"**으로 보냅니다.

*   **Normal Queue:** 정상 작업 처리
*   **DLQ:** 실패한 작업들이 모이는 곳. 개발자는 나중에 여기 쌓인 메시지를 보고 "아, 이래서 실패했구나" 분석하고, 버그를 고친 뒤 다시 Normal Queue로 넣어줍니다(Replay).

RabbitMQ에서는 `x-dead-letter-exchange` 옵션 하나로 쉽게 설정할 수 있습니다.

---

## 3. Idempotency (멱등성): 두 번 실행돼도 괜찮아?

가장 중요한 개념입니다.
네트워크 문제로 `Ack` 신호가 유실되면, RabbitMQ는 워커가 일을 안 한 줄 알고 **똑같은 메시지를 또 보냅니다(Redelivery).**

만약 그 작업이 "100만원 송금"이라면?
*   **Non-Idempotent:** 100만원이 두 번 빠져나감. (망함)
*   **Idempotent:** 두 번째 요청은 "어? 이미 처리된 ID네?" 하고 무시함. (안전)

### 구현 방법
모든 메시지에 고유한 `Message-ID` 혹은 `Idempotency-Key`를 부여하고, Redis 같은 곳에 처리 여부를 기록해야 합니다.

```python
key = f"processed:{message_id}"
if redis.exists(key):
    return "Already Done"

process_payment()
redis.set(key, "true")
```

---

## 🛠️ Lab: The Unstable Consumer

일부러 50% 확률로 실패하는 워커를 만들고, RabbitMQ의 DLQ 설정이 제대로 동작하는지 확인합니다.

1.  `main_queue`와 `dead_letter_queue`를 선언합니다.
2.  `main_queue`의 설정을 잡습니다: `arguments={"x-dead-letter-exchange": "", "x-dead-letter-routing-key": "dead_letter_queue"}`
3.  워커에서 메시지를 받을 때, `nack(requeue=False)`를 날려봅니다.
4.  메시지가 사라지지 않고 DLQ로 이동하는지 RabbitMQ Management UI에서 확인하세요.

---

## 📝 6주차 과제: Idempotent Payment Processor

**목표:** 멱등성이 보장된 가상 결제 시스템을 구현하세요.

1.  **Input:** `{ "transaction_id": "tx_123", "amount": 1000 }`
2.  **Logic:**
    *   Redis를 사용해 `transaction_id` 중복 체크.
    *   처음 들어온 ID면 "결제 성공" 로그 찍고 Redis에 저장.
    *   이미 있는 ID면 "중복 결제 방지됨" 로그 찍고 무시.
3.  **Test:** 동일한 JSON 데이터를 5번 연속으로 보내보세요. 실제 결제 로직은 **단 한 번**만 실행되어야 합니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** "멱등성(Idempotency)"이란 무엇인가요? (연산을 여러 번 적용해도 결과가 달라지지 않는 성질)
2.  **Q2.** 지수 백오프(Exponential Backoff)를 사용할 때, 모든 클라이언트가 동시에 재시도하는 것을 막기 위해 추가하는 랜덤 값은? (Jitter)
3.  **Q3.** RabbitMQ에서 메시지가 DLQ로 이동하는 조건 3가지는? (Nack/Reject, TTL 만료, Queue 길이 초과)

---

다음 주에는 이러한 복잡한 Retry, DLQ 로직을 직접 짜지 않고도 쓸 수 있게 해주는 도구, **Celery**를 만납니다.

**Fail gracefully.**
