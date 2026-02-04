---
layout: post
title: "Backend Architecture Mastery: Week 10 - Caching Strategies at Scale"
---

DB 쿼리가 느리다고요? 인덱스를 태웠는데도 느리다고요?
그렇다면 답은 하나입니다. **"안 읽으면 됩니다."**

메모리(RAM)는 디스크보다 10만 배 빠릅니다. 자주 찾는 데이터를 Redis 같은 In-Memory Cache에 올려두는 것, 그것이 성능 최적화의 끝판왕입니다. 하지만 잘못 쓰면 독이 됩니다.

---

## 1. Cache Patterns

### Cache-Aside (Lazy Loading)
가장 보편적인 패턴입니다.
1.  앱이 캐시(Redis)를 찌릅니다.
2.  **Hit:** 있으면 그거 줍니다. (DB 안 감. 개이득)
3.  **Miss:** 없으면 DB에서 읽어옵니다. 그리고 캐시에 저장(`Set`)하고 줍니다.

### Write-Through / Write-Back
*   **Write-Through:** DB에 쓸 때 캐시에도 같이 씀. (데이터 일관성 좋음, 쓰기 느림)
*   **Write-Back:** 캐시에만 먼저 쓰고, 나중에 DB에 몰아서 씀. (엄청 빠름, 캐시 죽으면 데이터 날아감)

---

## 2. The Cache Problems (면접 단골)

### Thundering Herd (캐시 쇄도)
인기 있는 게시글의 캐시가 만료(`TTL Expire`)되었습니다.
그 순간 1만 명이 동시에 접속하면? 1만 개의 요청이 동시에 DB를 때립니다. DB는 즉사합니다.
*   **해결:** 캐시 만료 시간을 랜덤하게 분산시키거나(Jitter), 한 명만 DB에 가도록 락을 겁니다.

### Cache Penetration (캐시 관통)
악의적인 해커가 DB에 없는 ID(`id=-1`)로 계속 요청합니다.
캐시엔 당연히 없으니 계속 DB를 찌르게 됩니다.
*   **해결:** "없음(Null)"이라는 사실 자체를 캐싱합니다. (TTL 짧게)

---

## 3. Redis in Python

`redis-py` 혹은 비동기 버전인 `redis-om`을 사용합니다.

```python
import redis.asyncio as redis
import json

r = redis.Redis(host='localhost', port=6379, db=0)

async def get_user(user_id: int):
    # 1. Cache Look up
    cache_key = f"user:{user_id}"
    cached = await r.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # 2. DB Look up
    user = db.query(User).get(user_id)
    
    # 3. Save to Cache (TTL 10분)
    await r.setex(cache_key, 600, json.dumps(user.dict()))
    return user
```

---

## 🛠️ Lab: Redis as a Shared State

분산 환경에서 Redis는 단순 캐시가 아니라 **공유 메모리** 역할을 합니다.
FastAPI 인스턴스 3개를 띄우고, Rate Limiter(요청 제한)를 구현해 봅니다.

1.  User IP를 키로 하여 카운터를 올립니다. (`INCR`)
2.  카운터가 1일 때 `EXPIRE`를 60초로 줍니다.
3.  카운터가 10을 넘으면 429 Error를 뱉습니다.

어떤 인스턴스로 요청이 가든, Redis 카운터는 공유되므로 정확하게 차단됩니다.

---

## 📝 10주차 과제: Leaderboard System

**목표:** Redis의 **Sorted Set** 자료구조를 이용하여 실시간 랭킹 시스템을 구현하세요.

1.  `ZADD leaderboard <score> <user_id>`: 점수 등록
2.  `ZREVRANGE leaderboard 0 9 WITHCORES`: Top 10 조회
3.  RDBMS(SQL)로 구현했을 때와 Redis로 구현했을 때의 속도 차이(Big O Notation 관점)를 서술하세요. (SQL: O(N log N) vs Redis: O(log N))

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 캐시가 가득 찼을 때, 가장 오랫동안 사용되지 않은 데이터를 지우는 알고리즘은? (LRU - Least Recently Used)
2.  **Q2.** DB 데이터가 변경되었을 때 캐시 데이터를 즉시 갱신하지 않고 만료될 때까지 기다리는 방식을 무엇이라 하나요? (Eventual Consistency)
3.  **Q3.** Redis는 싱글 스레드인가요 멀티 스레드인가요? (싱글 스레드 - 그래서 Atomic 연산이 보장됨)

---

다음 주, 클라이언트와 서버가 실시간으로 수다를 떠는 **WebSockets**의 세계로 갑니다.
새로고침(F5)은 이제 그만.

**Cache me if you can.**
