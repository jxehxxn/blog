---
layout: post
title: "Backend Architecture Mastery: Week 9 - Database Concurrency (Locks & ACID)"
---

"재고가 1개 남았는데, 100명이 동시에 구매 버튼을 눌렀습니다. 몇 명이 구매에 성공할까요?"

개발자가 흔히 하는 착각은 "DB가 알아서 해주겠지"입니다.
하지만 기본 설정에서 대부분의 DB는 **동시성 이슈(Race Condition)**에 취약합니다. 운이 나쁘면 재고는 -99개가 될 수도 있습니다.

오늘은 데이터의 무결성을 지키는 방패, **Transaction Isolation Level**과 **Lock**에 대해 배웁니다.

---

## 1. The Race Condition

```python
# 잘못된 코드 예시
item = db.query(Item).filter(Item.id == 1).first()
if item.stock > 0:
    item.stock -= 1
    db.commit()
```
이 코드는 스레드 안전하지 않습니다. `SELECT` 하고 `UPDATE` 하는 그 찰나의 순간에 다른 트랜잭션이 끼어들 수 있기 때문입니다. 이를 **Lost Update** 문제라고 합니다.

---

## 2. Pessimistic Lock vs Optimistic Lock

### Pessimistic Lock (비관적 락)
"누군가 내 데이터를 건드릴 것 같아. 아예 문을 잠그자."
*   **SQL:** `SELECT * FROM items WHERE id = 1 FOR UPDATE`
*   **동작:** 트랜잭션이 끝날 때까지 해당 행(Row)을 물리적으로 잠급니다. 다른 놈은 읽지도 못하게 하죠.
*   **장점:** 데이터 정합성 확실함.
*   **단점:** 성능 저하, 데드락(Deadlock) 위험.

### Optimistic Lock (낙관적 락)
"설마 겹치겠어? 일단 읽고, 저장할 때 버전 확인해보지 뭐."
*   **구현:** `version` 컬럼을 추가합니다.
*   **Logic:** `UPDATE items SET stock = stock - 1, version = version + 1 WHERE id = 1 AND version = 5`
*   **결과:** 만약 그 사이 누가 버전을 6으로 올렸으면, 이 쿼리는 0 rows affected가 됩니다. 그때 앱에서 재시도(Retry)하거나 에러를 냅니다.
*   **장점:** 락을 안 거니 빠름.
*   **단점:** 충돌이 잦으면 재시도 비용이 더 큼.

---

## 3. Isolation Levels (격리 수준)

DB가 어디까지 막아줄 거냐의 설정입니다. (MySQL InnoDB 기준)
1.  **READ UNCOMMITTED:** 똥밭. 커밋 안 된 데이터도 읽힘 (Dirty Read). 절대 쓰지 마세요.
2.  **READ COMMITTED:** 커밋된 것만 읽힘. (PostgreSQL 기본값)
3.  **REPEATABLE READ:** 트랜잭션 내에서는 항상 같은 데이터가 보임. (MySQL 기본값)
4.  **SERIALIZABLE:** 완벽한 줄 세우기. 느림.

---

## 🛠️ Lab: The Inventory Crash

FastAPI와 SQLAlchemy를 이용해 동시성 문제를 재현하고, **비관적 락(`with_for_update`)**으로 해결해 봅니다.

1.  `stock = 1`인 아이템을 만듭니다.
2.  `asyncio.gather`를 사용해 동시에 10번 구매 요청을 보냅니다.
3.  락이 없을 때: 재고가 음수가 되는 것을 확인.
4.  락 적용 후: 1명만 성공하고 9명은 대기하다가 재고 부족 에러가 나는 것을 확인.

```python
# 올바른 예시 (Pessimistic Lock)
stmt = select(Item).where(Item.id == item_id).with_for_update()
item = await session.execute(stmt)
# ...
```

---

## 📝 9주차 과제: Bank Transfer System

**목표:** A계좌에서 B계좌로 돈을 이체하는 API를 작성하세요. 단, **데드락(Deadlock)**을 방지해야 합니다.

1.  **상황:** A가 B에게 100원 송금, 동시에 B가 A에게 100원 송금.
2.  잘못 짜면 서로의 계좌에 락을 걸고 영원히 기다립니다.
3.  **해결책:** 항상 `id` 순서대로 락을 거는 등 데드락 회피 전략을 적용하세요. (e.g., 작은 ID 먼저 락)

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 비관적 락을 걸기 위해 사용하는 SQL 구문은? (SELECT ... FOR UPDATE)
2.  **Q2.** 낙관적 락 구현을 위해 테이블에 반드시 추가해야 하는 컬럼은? (version 또는 timestamp)
3.  **Q3.** 트랜잭션의 4가지 원칙(ACID) 중, 트랜잭션이 모두 수행되거나 아예 수행되지 않아야 함을 의미하는 것은? (Atomicity - 원자성)

---

다음 주, DB 부하를 줄여주는 마법의 도구 **Cache(Redis)**에 대해 배웁니다.
"There are only two hard things in Computer Science: cache invalidation and naming things."

**Lock it down.**
