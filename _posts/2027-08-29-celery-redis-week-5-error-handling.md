---
layout: post
title: "Distributed Task Processing with Celery & Redis: Week 5 - Error Handling & Retry Strategies"
---

"외부 API가 500 에러를 뱉어요."
"DB 커넥션이 일시적으로 끊겼어요."

분산 시스템에서 에러는 **일상(Routine)**입니다.
따라서 Task는 실패했을 때 우아하게 **재시도(Retry)**해야 합니다.
하지만 무작정 재시도하면 서버를 더 죽입니다. **Exponential Backoff**가 필요한 이유입니다.

---

## 1. Basic Retry

`try-except` 블록 안에서 `self.retry()`를 호출합니다.

```python
@app.task(bind=True)
def send_email(self, user_id):
    try:
        smtp.send(...)
    except SMTPConnectError as e:
        # 10초 뒤에 재시도. 최대 3번.
        raise self.retry(exc=e, countdown=10, max_retries=3)
```

**주의:** `raise`를 꼭 붙여야 합니다. `retry()` 메서드는 특수한 예외를 발생시켜서 Task 실행을 중단시키기 때문입니다.

---

## 2. Automatic Retry (`autoretry_for`)

코드를 더 깔끔하게 짜고 싶다면 데코레이터 옵션을 씁니다.

```python
@app.task(autoretry_for=(SMTPConnectError,), retry_kwargs={'max_retries': 5}, retry_backoff=True)
def send_email(user_id):
    smtp.send(...)
```
이러면 `try-except` 블록 없이도 `SMTPConnectError`가 나면 알아서 재시도합니다.

---

## 3. Exponential Backoff (지수 백오프)

서버가 과부하로 죽어가는데 1초마다 계속 재시도하면 확인사살 하는 꼴입니다.
점점 기다리는 시간을 늘려야 합니다.

*   1회차: 1초 대기
*   2회차: 2초 대기
*   3회차: 4초 대기
*   4회차: 8초 대기...

Celery에서는 `retry_backoff=True` 옵션 하나면 끝입니다.
`retry_backoff_max`로 최대 대기 시간을 제한할 수도 있습니다.

---

## 4. Dead Letter Queue (DLQ)

재시도를 5번이나 했는데도 실패하면?
그냥 버리면 안 됩니다. **"왜 실패했는지"** 분석하기 위해 따로 보관해야 합니다.

Celery는 기본적으로 실패한 작업 정보를 저장하지 않습니다(Result Backend 제외).
하지만 RabbitMQ를 쓴다면 DLX(Dead Letter Exchange) 설정을 통해 실패한 메시지를 별도 큐로 뺄 수 있습니다.
Redis를 쓴다면... 아쉽게도 Celery 레벨에서는 직접 구현하거나 `on_failure` 핸들러에서 DB에 기록해야 합니다.

```python
class MyTask(Task):
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        # DB에 실패 기록 저장
        FailedTask.objects.create(task_id=task_id, error=str(exc))

@app.task(base=MyTask)
def add(x, y):
    ...
```

---

## 🛠️ Lab: The Flaky API

랜덤하게 실패하는 Task를 만들고 Retry 동작을 관찰합니다.

1.  **Task 정의:** 50% 확률로 `ValueError`를 발생시키는 Task.
2.  **설정:** `max_retries=3`, `retry_backoff=True`.
3.  **실행:** 워커 로그를 켜놓고(`-l info`) Task를 여러 번 호출합니다.
4.  **관찰:**
    *   "Retry in 1s..."
    *   "Retry in 2s..."
    *   "Max retries exceeded" 에러가 뜨는지 확인.

---

## 📝 5주차 과제: Custom Backoff Strategy

**목표:** 지수 백오프 대신 **Jitter(랜덤값)**가 포함된 커스텀 재시도 로직을 구현하세요.

**이유:** 수만 개의 작업이 동시에 실패했다가 동시에 재시도하면(Thundering Herd), 서버가 또 죽습니다. 재시도 시간을 분산시켜야 합니다.

1.  `self.retry(countdown=...)` 파라미터를 동적으로 계산하세요.
2.  공식: `(2 ** self.request.retries) + random.uniform(0, 1)`
3.  로그에 찍히는 대기 시간이 매번 조금씩 다른지 확인하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** `self.retry()` 호출 시 `raise` 키워드를 함께 사용해야 하는 이유는? (retry 함수가 예외를 발생시켜 현재 Task의 실행 흐름을 중단하고 예외 처리를 Celery에게 위임하기 위해)
2.  **Q2.** 서버 부하를 줄이기 위해 재시도 간격을 점진적으로 늘리는 기법을 무엇이라 하나요? (Exponential Backoff)
3.  **Q3.** 모든 재시도가 실패했을 때 호출되는 Celery Task의 콜백 메서드는? (on_failure)

---

다음 주, 작업 하나로는 부족합니다.
"A 하고, 그 결과로 B 하고, 동시에 C도 해라."
복잡한 워크플로우를 그리는 **Celery Canvas**를 배웁니다.

**Fail gracefully.**
