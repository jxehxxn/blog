---
layout: post
title: "Distributed Task Processing with Celery & Redis: Supplement - Redis Persistence (RDB vs AOF) for Queues"
---

Week 4에서 Redis Persistence를 다뤘습니다. 하지만 "브로커로서의 Redis"에 최적화된 설정은 일반적인 캐시용 설정과 다릅니다.

이 보충 포스트에서는 RDB와 AOF의 작동 원리를 그림 그리듯 이해하고, **작업 큐(Message Queue)를 위한 최적의 `redis.conf`**를 제안합니다.

---

## 1. RDB (Redis Database): The Snapshot

RDB는 특정 시점의 메모리 전체를 사진 찍듯 파일(`dump.rdb`)로 저장합니다.

### Trigger Point (기본값)
```conf
save 900 1     # 15분 동안 1개 키 변경 시
save 300 10    # 5분 동안 10개 키 변경 시
save 60 10000  # 1분 동안 1만 개 키 변경 시
```

### Queue에서의 문제점
작업 큐는 **매우 빈번하게 쓰고 지워집니다.** (Task Enqueue -> Dequeue)
1초에 1000개의 작업이 처리된다면?
데이터는 엄청 변하는데, `save 60 10000` 조건 때문에 1분마다 저장됩니다.
**재수 없게 59초에 전원이 나가면? 59초 동안 처리된 작업 상태나 대기 중인 작업 59,000개가 날아갑니다.**

---

## 2. AOF (Append Only File): The Log

AOF는 명령어를 하나하나 기록합니다. (`LPUSH ...`, `DEL ...`)

### Sync Strategy (fsync)
*   `appendfsync always`: 명령마다 디스크에 씀. (데이터 유실 0, 속도 매우 느림)
*   `appendfsync everysec`: **1초마다** 디스크에 씀. (최대 1초 유실, 속도 빠름) -> **권장**
*   `appendfsync no`: OS한테 맡김. (빠르지만 위험)

### Rewrite (압축)
로그가 계속 쌓이면 파일이 수백 GB가 됩니다.
`LPUSH task1`, `DEL task1` -> 결과적으로 아무것도 없으므로 로그에서 삭제.
Redis는 주기적으로 이 작업을 수행하여 파일 크기를 줄입니다.

---

## 3. Best Practice for Celery Broker

Celery Broker 용도로 Redis를 쓴다면 다음 설정을 권장합니다.

### 1) Persistence
**AOF (`appendfsync everysec`)를 켜세요.**
RDB는 끄거나 백업용으로만 쓰세요. 작업 큐는 데이터 변경이 너무 잦아서 RDB로는 유실을 막기 어렵습니다.

```conf
appendonly yes
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

### 2) Maxmemory Policy
메모리가 꽉 찼을 때 어떻게 할 것인가?
캐시라면 `allkeys-lru` (오래된 거 지움)가 좋지만, **큐는 절대 지우면 안 됩니다.**

```conf
maxmemory-policy noeviction
```
이렇게 하면 메모리가 꽉 찼을 때 에러를 뱉습니다. (OOM). 작업을 잃어버리는 것보다 낫습니다. 이때는 모니터링 알람을 받고 워커를 늘려야 합니다.

---

## 4. Summary

| 특징 | RDB (Snapshot) | AOF (Log) |
| :--- | :--- | :--- |
| **속도** | 빠름 (저장할 때만 Fork) | 약간 느림 (계속 씀) |
| **데이터 유실** | 마지막 저장 이후 다 날아감 | 최대 1초 (everysec 기준) |
| **복구 속도** | 빠름 (파일 로드) | 느림 (명령어 재실행) |
| **큐 적합성** | 낮음 | **높음** |

"Redis는 원래 날아가는 거니까"라고 포기하지 마세요. 설정만 잘하면 꽤 튼튼한 브로커가 됩니다.
