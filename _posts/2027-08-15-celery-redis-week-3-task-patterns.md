---
layout: post
title: "Distributed Task Processing with Celery & Redis: Week 3 - Task Design Patterns (Idempotency, Atomicity)"
---

분산 시스템의 제1법칙: **"네트워크는 믿을 수 없다."**
워커가 작업을 하다가 전원이 꺼질 수도 있고, Broker가 응답을 못 받을 수도 있습니다.
이때 Celery는 작업을 **재시도(Retry)**합니다.

만약 그 작업이 "결제"라면? 1번 결제했는데 에러 난 줄 알고 또 결제하면?
이런 참사를 막기 위해 **멱등성(Idempotency)**과 **원자성(Atomicity)**을 반드시 이해해야 합니다.

---

## 1. Idempotency (멱등성)

수학 용어지만 개념은 간단합니다.
**"연산을 여러 번 적용하더라도 결과가 달라지지 않는 성질."**

*   `f(x) = x + 1`: 멱등성 없음. 2번 실행하면 2가 더해짐.
*   `f(x) = set(x, 5)`: 멱등성 있음. 100번 실행해도 x는 5임.

### Celery Task에서의 멱등성
작업 큐에서는 **"At Least One Delivery"** (최소 한 번은 전달됨)가 기본입니다. 즉, **두 번 전달될 수도 있다**는 뜻입니다.
따라서 모든 Task는 두 번 실행되어도 안전하게 설계해야 합니다.

**Bad:**
```python
@app.task
def give_bonus(user_id):
    user = User.get(user_id)
    user.points += 100  # 위험! 두 번 실행되면 200원 줌.
    user.save()
```

**Good:**
```python
@app.task
def give_bonus(user_id, transaction_id):
    if Transaction.exists(transaction_id):
        return "Already Processed"
    
    user = User.get(user_id)
    user.points += 100
    Transaction.create(id=transaction_id) # 처리 기록 남김
    user.save()
```

---

## 2. Atomicity (원자성)

**"전부 다 실행되거나, 아예 실행되지 않거나."**
중간에 멈추면 안 됩니다.

**Scenario:**
1.  사용자 포인트 차감.
2.  (여기서 워커 죽음)
3.  기프트콘 발송.

이렇게 되면 포인트만 날아갑니다. DB 트랜잭션(`transaction.atomic()`)을 사용하여 이 두 과정을 하나로 묶어야 합니다.

---

## 3. The "Late Acknowledgment" Problem

Celery는 기본적으로 작업이 **완료된 후**에 Broker에게 ACK를 보냅니다(`acks_late=False`가 기본이지만 명시적으로 늦게 보내기도 함).
*   작업 도중 워커가 죽으면 -> ACK가 안 감 -> Broker는 다른 워커에게 재전송 -> 작업 재실행.

이때 멱등성이 없으면 재앙이 일어납니다.
따라서 **"멱등성을 보장할 수 없다면, 차라리 작업이 유실되는 게 낫다(At Most Once)"**는 전략을 취하기도 합니다. (`acks_late=False` 유지)

---

## 🛠️ Lab: Implementing Idempotent Task

Redis를 활용해 중복 실행을 방지하는 락(Lock) 패턴을 구현해 봅니다.

```python
from django.core.cache import cache

@app.task(bind=True)
def send_welcome_email(self, user_id):
    lock_id = f"email-lock-{user_id}"
    
    # 1. 락 획득 시도 (Atomic 연산)
    # acquire_lock은 redis.setnx()를 사용
    acquire_lock = lambda: cache.add(lock_id, "true", timeout=60*5)
    
    if acquire_lock():
        try:
            # 2. 이메일 전송 로직
            send_email(user_id)
        finally:
            # 3. 락 해제? No! 
            # 멱등성을 위해 락을 해제하지 않고 만료되게 두는 패턴도 있음.
            # (한 번 보냈으면 5분 안에는 절대 다시 안 보냄)
            pass
    else:
        print("Duplicate task skipped.")
```

---

## 📝 3주차 과제: Design an Order Processing Task

**목표:** 주문 처리 Task를 설계하세요.

**요구사항:**
1.  Task 인자: `order_id`
2.  로직:
    *   재고 차감
    *   결제 승인
    *   배송 요청
3.  **문제 상황:** 결제 승인까지 했는데 배송 요청에서 외부 API 에러가 나서 Task가 재시도(Retry) 되었습니다.
4.  **해결책:** 재고 차감과 결제 승인이 **두 번 일어나지 않도록** 방어 코드를 작성하세요. (Pseudo-code 가능)

---

## ✅ Self-Assessment Quiz

1.  **Q1.** "동일한 요청을 여러 번 보내도 결과가 같다"는 성질을 무엇이라 하나요? (Idempotency - 멱등성)
2.  **Q2.** Celery 기본 설정에서 Worker가 작업을 수행하다가 강제 종료되면, 그 작업은 어떻게 되나요? (Broker가 Timeout을 감지하고 다른 Worker에게 재할당함 - Visibility Timeout)
3.  **Q3.** 여러 개의 DB 연산을 하나의 단위로 묶어서, 하나라도 실패하면 전체를 롤백(Rollback)시키는 속성은? (Atomicity - 원자성)

---

다음 주, 우리가 믿고 쓰는 **Redis**가 사실 브로커로서는 어떤 한계가 있는지, 그리고 어떻게 하면 더 잘 쓸 수 있는지 **Redis Deep Dive**를 진행합니다.

**Design for failure.**
