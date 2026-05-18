---
layout: post
title:  "Dreamhack 워게임 PATCH-1 풀이 — 한 화면에 SQLi, SSTI, 약한 비밀키, 권한 검증 누락이 모두 들어 있는 패치 챌린지"
date:   2026-05-19 01:00:00 +0900
categories: security web wargame dreamhack writeup
---

## 들어가며

이번 글에서는 Dreamhack 웹해킹 워게임 **PATCH-1** (난이도 Silver 1) 문제를 정보보안 입문자의 시점에서 풀어봅니다. 이름 그대로 _patch_ 문제 — **취약점을 _찾는_ 것이 아니라 _고치는_ 것** 이 목표인 챌린지입니다. 일반적인 워게임이 "공격 페이로드 한 줄을 찾으면 끝" 이라면, 패치 챌린지는 _공격자의 머리_ 와 _방어자의 손_ 을 동시에 요구합니다.

문제 설명은 단 세 줄입니다.

> 주어진 코드를 분석하고, 해당 코드에 존재하는 취약점들을 패치해보세요.
> 문제에 대한 자세한 설명은 /usage 페이지를 확인하여 보시기 바랍니다.
> 모든 패치가 완료되면 플래그를 획득할 수 있습니다.

`/usage` 의 규칙은 다음과 같습니다.

- _**수정 가능 표시("✔ 수정 가능")가 있는 파일만 수정**_ 가능. 본 문제에선 `app.py` 하나.
- 서버는 _SLA(기능 동작 검증)_ 와 _보안 검증_ 을 함께 돌린다. _둘 다 통과_ 해야 플래그.
- _**제출 간격 30초**_. 즉 _시도-실패-수정_ 사이클이 비싸다.
- `socket`/`execve` 같은 외부 호출 금지, `os.environ` 수정 금지, `app.run` 추가 금지.

> Spoiler: 결국 _네 개의 취약점_ 을 한 번에 잡으면 ALL PASS 가 나옵니다. 코드 자체가 짧기 때문에 분석은 5분이면 끝나지만, _**SLA를 깨뜨리지 않으면서 보안만 고치는 균형**_ 이 이 문제의 진짜 학습 포인트입니다.

## 1. 문제 파일 들여다보기 — `app.py` 한 줄 한 줄 읽기

ZIP을 풀면 `app.py` 단 한 개. 100라인 정도의 Flask + SQLite3 API 서버. 그대로 옮겨 적으면서 _**한 핸들러씩 적신호를 표시**_ 해 봅니다.

### 1-1. 초기 설정 — _하드코딩 비밀키_

```python
app = Flask(__name__)
app.secret_key = "Th1s_1s_V3ry_secret_key"
```

**🚩 1번 적신호** : Flask 세션은 _서명(sign)_ 만 되고 _암호화(encrypt)_ 는 안 됩니다 (이 사실은 [Secure Secret 풀이](/blog/2026/05/18/dreamhack-secure-secret-writeup/) 에서 자세히 다뤘습니다). 그 _서명 키_ 가 **소스 코드에 그대로 박혀 있고**, **공격자도 동일한 ZIP을 다운로드 받아서 그 키를 알고 있습니다**. 즉, _임의의 세션 위조_ 가 가능합니다 — `session['uid'] = 'admin'` 같은 쿠키를 직접 만들어 서버에 보내면 _서명 검증 통과_.

> 공격 예시 (`flask-unsign` 도구로 한 줄):
> ```
> flask-unsign --sign --cookie '{"uid": "admin"}' --secret 'Th1s_1s_V3ry_secret_key'
> ```

**패치 방향** : 키를 _런타임 무작위 값_ 으로 만든다. `os.urandom(32).hex()` 가 가장 단순한 해법.

### 1-2. `/api/login` — _고전적 SQL injection_

```python
ret = query_db(f"SELECT * FROM users where userid='{userid}' "
               f"and password='{hashlib.sha256(password.encode()).hexdigest()}'",
               one=True)
```

**🚩 2번 적신호** : `userid` 가 _f-string_ 으로 _직접 SQL 안에 결합_. 비밀번호는 sha256 해시로 한 번 가공되지만, _ID는 raw_ 입니다. 페이로드:

```text
userid: admin' --
password: anything
```

→ `SELECT * FROM users where userid='admin' --' and password='...'`. `--` 이후가 주석 처리되어 _비밀번호 체크 없이_ admin 로그인 성공.

**패치 방향** : `?` 자리표시자 + 튜플 인자.

```python
ret = query_db(
    "SELECT * FROM users where userid=? and password=?",
    (userid, hashlib.sha256(password.encode()).hexdigest()),
    one=True,
)
```

### 1-3. `/api/memo/<idx>` GET — _또 다른 SQL injection_

```python
@app.route('/api/memo/<idx>', methods=['GET'])
def memoView(idx):
  ret = query_db("SELECT * FROM memo where idx=" + idx)[0]
```

**🚩 3번 적신호** : URL 경로 변수 `<idx>` 가 _문자열 연결_ 로 SQL 에 박힘. `<int:idx>` 같은 _타입 컨버터_ 도 쓰지 않아서, 라우터 입장에선 _임의 문자열_ 입니다. 페이로드:

```
GET /api/memo/1%20UNION%20SELECT%20userid,password,userid,1%20FROM%20users
```

→ 다른 사용자 비밀번호 해시까지 뽑힘.

**패치 방향** : 똑같이 `?` 자리표시자.

```python
ret = query_db("SELECT * FROM memo where idx=?", (idx,))[0]
```

### 1-4. `/api/memo/<idx>?mode=html` — _Server-Side Template Injection (SSTI)_

```python
if mode == 'html':
    template = ''' Written by {userid}<h3>{title}</h3>
    <pre>{contents}</pre>
    '''.format(title=title, userid=userid, contents=contents)
    return render_template_string(template)
```

**🚩 4번 적신호** : 이게 가장 미묘합니다. 두 단계가 결합하면서 폭발합니다.

1. `.format(title=title, userid=userid, contents=contents)` 가 _파이썬 str.format_ 으로 `{title}` 등을 _**그대로 사용자 입력으로 치환**_ 합니다.
2. 치환된 결과를 _**`render_template_string` 으로 Jinja2 템플릿으로 다시 렌더링**_ 합니다.

따라서 사용자가 `title=` 에 _Jinja2 문법_ 을 넣으면, _첫 단계에서 그대로 박힌 후_ 두 번째 단계에서 _Jinja2가 평가_ 합니다. 가장 단순한 SSTI 페이로드:

```text
title: {{7*7}}
```

→ 결과 HTML 에 `49` 출력. 이는 곧 **임의 파이썬 코드 실행** 으로 확장 가능합니다. Jinja2 의 클래식한 RCE 가젯:

```text
{{ request.application.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

또는 `{% set ... %}`, `{{ ''.__class__.__mro__[1].__subclasses__() }}` 류.

**패치 방향** : _**사용자 데이터를 템플릿 _문자열_ 에 박지 말고, 템플릿 _변수_ 로 전달**_ 한다.

```python
if mode == 'html':
    template = ''' Written by {{userid}}<h3>{{title}}</h3>
    <pre>{{contents}}</pre>
    '''
    return render_template_string(template, title=title, userid=userid, contents=contents)
```

핵심 차이는 두 가지입니다.

- `.format()` 으로 미리 박지 _않는다_ — 사용자 입력이 _**템플릿의 일부가 되지 못하게**_.
- 템플릿 안의 _자리_ 는 Jinja2 의 `{{ ... }}` 로 표기.

이렇게 하면 `{{title}}` 안에 들어가는 값이 _자동으로 HTML escape_ 되며 _코드로 평가되지 않습니다_ (Jinja2 의 autoescape 정책 + 변수는 _문자열_ 로 취급).

### 1-5. `/api/memo/<int:idx>` PUT — _권한 검증 누락_

```python
@app.route('/api/memo/<int:idx>', methods=['PUT'])
def memoUpdate(idx):
  if not session.get('uid'):
    return jsonify(result="no login")
  ret = query_db('SELECT * FROM memo where idx=?', [idx,])[0]
  userid = session.get('uid')
  title = request.form.get('title')
  contents = request.form.get('contents')

  if ret and title and contents:
    # ... UPDATE memo SET title=?, contents=? WHERE idx=?
```

**🚩 5번 적신호** : 로그인 여부만 확인할 뿐, _**조회된 메모의 작성자(`ret['userid']`)와 현재 세션 사용자(`session['uid']`)가 일치하는지 검증하지 않습니다**_. 따라서 _**아무 로그인 사용자가 다른 사람의 메모를 자유롭게 수정**_ 할 수 있는 _IDOR/권한 검증 누락_.

**패치 방향** : 한 줄 조건 추가.

```python
if ret and title and contents and ret['userid'] == userid:
    # ... UPDATE ...
```

> 일부 PATCH-1 풀이에서는 이 권한 검증 누락을 _**SLA가 검사하지 않는다**_ 고 보고 패치를 생략하기도 합니다. 그러나 _**보안 관점에서 명백한 결함**_ 이므로, ALL PASS 가 안 나오면 _자기 책임으로_ 추가하지 않아도 통과한다는 의미일 뿐이지, _고치지 않는 것이 정답인 것은 아닙니다_. 본 풀이에서는 _현실 코드에 가까운 정답_ 을 위해 함께 패치했습니다.

## 2. 패치 — 최종 `app.py`

이 다섯 줄(secret_key + SQLi×2 + SSTI + IDOR) 을 동시에 고친 결과:

```python
#!/usr/bin/python3
from flask import Flask, request, render_template_string, g, session, jsonify
import sqlite3
import os, hashlib

app = Flask(__name__)
app.secret_key = os.urandom(32).hex()                                 # (1) random key

# ... (get_db, query_db, close_connection — 변경 없음)

@app.route('/api/login', methods=['POST'])
def login():
  userid = request.form.get('userid', '')
  password = request.form.get('password', '')
  if userid and password:
    ret = query_db(                                                    # (2) parameterized
      "SELECT * FROM users where userid=? and password=?",
      (userid, hashlib.sha256(password.encode()).hexdigest()),
      one=True,
    )
    if ret:
      session['uid'] = ret[0]
      return jsonify(result="success", userid=ret[0])
  return jsonify(result="fail")

@app.route('/api/memo/<idx>', methods=['GET'])
def memoView(idx):
  mode = request.args.get('mode', 'json')
  ret = query_db("SELECT * FROM memo where idx=?", (idx,))[0]          # (3) parameterized
  if ret:
    userid = ret['userid']
    title = ret['title']
    contents = ret['contents']
    if mode == 'html':
      template = ''' Written by {{userid}}<h3>{{title}}</h3>            ## (4) Jinja vars
      <pre>{{contents}}</pre>
      '''
      return render_template_string(template, title=title, userid=userid, contents=contents)
    else:
      return jsonify(result="success", userid=userid, title=title, contents=contents)
  return jsonify(result="error")

@app.route('/api/memo/<int:idx>', methods=['PUT'])
def memoUpdate(idx):
  if not session.get('uid'):
    return jsonify(result="no login")
  ret = query_db('SELECT * FROM memo where idx=?', [idx,])[0]
  userid = session.get('uid')
  title = request.form.get('title')
  contents = request.form.get('contents')
  if ret and title and contents and ret['userid'] == userid:           # (5) ownership check
    conn = get_db(); cur = conn.cursor()
    updateRet = cur.execute("UPDATE memo SET title=?, contents=? WHERE idx=?",
                            (title, contents, idx))
    conn.commit()
    if updateRet:
      return jsonify(result="success")
  return jsonify(result="error")
```

(나머지 핸들러 `me`, `logout`, `join`, `memoAdd` 는 원본 그대로. 이미 안전합니다.)

## 3. 제출 — `curl` 두 번이면 끝

문제 페이지의 _Submit_ 버튼은 _패치된 코드를 base64로 인코딩해서_ `/check` 에 POST 합니다. `/check` 는 _diff 미리보기_ 와 _`/run` 으로 가는 form_ 을 돌려줍니다. 사용자가 _Continue_ 를 누르면 `/run` 으로 다시 POST되고, _30초 cooldown_ 안에 _실제 채점_ 이 진행됩니다.

```bash
URL=http://host8.dreamhack.games:13177
JAR=/tmp/p1_cookie.txt
B64=$(base64 -i patched_app.py | tr -d '\n')

# 1) /check — 세션 쿠키 받기 + diff 확인
curl -s -c "$JAR" -b "$JAR" -X POST --data-urlencode "app.py=$B64" "$URL/check" > /dev/null

# 2) 30초 대기 후 /run — 실제 채점
sleep 32
curl -s -b "$JAR" -X POST --data-urlencode "app.py=$B64" "$URL/run" -i | grep Location
# → Location: http://.../result/<uuid>

# 3) 결과 페이지에서 ALL PASS + 플래그 확인
curl -s -b "$JAR" "$URL/result/<uuid>" | grep -oE 'ALL PASS|DH\{[^}]+\}'
# → ALL PASS
# → DH{7e07d530518d7983c066a7019b6cf027}
```

ALL PASS 가 나오면 같은 페이지에 _**플래그 + 입력 코드**_ 가 모두 표시됩니다.

## 4. 한 발 더 — 각 취약점의 _**일반적**_ 패턴

각 취약점은 _이 문제만의 함정_ 이 아니라 _현업에서 흔히 보는_ 패턴입니다.

### 4-1. SQL injection — _f-string은 SQL의 적_

- _"이 변수는 안전하다"_ 는 _개발자의 가정_ 은 _리뷰어/공격자에게는 거짓_ 으로 들립니다.
- _**ORM**_ 을 쓰면 거의 자동으로 해결되지만, _raw SQL_ 을 쓰는 곳에선 **f-string / `+` 결합 / `%` 포매팅 모두 SQL injection의 직접 신호** 입니다.
- 라이브러리별 안전한 패턴:
  - `sqlite3` / `psycopg2` : `?` 자리표시자 (DB-API)
  - `SQLAlchemy Core` : `text("... :name")` + `bindparams`
  - `SQLAlchemy ORM` : 그냥 `.filter(User.name == userid)`
- _**금지**_ 패턴: `f"... {userid} ..."`, `"... " + userid + " ..."`, `query % (userid,)`.

### 4-2. SSTI — _포맷팅과 렌더링이 _만나면_ 폭발한다_

이 문제의 SSTI는 _Jinja2 자체의 결함_ 이 아니라, _**파이썬 `str.format` 결과를 다시 Jinja2 템플릿으로 넣는**_ 비정상 패턴 때문에 생겼습니다. 두 단계 _**문자열 처리**_ 가 결합될 때 _반드시 의심_ 해야 합니다. 보통 이렇게 깨집니다:

- `markdown.render(user_input)` → `render_template_string(html)` (HTML이 다시 Jinja로 들어감)
- 이메일 템플릿: `subject = f"Hello {name}"` → 메일 라이브러리가 _이 문자열을 다시 템플릿으로 평가_
- 다국어: `gettext(key) % user_input` → `render_template_string(...)` 으로 출력

_방어 원칙_: **사용자 데이터는 _**템플릿의 _데이터_ 부분_** 으로만 들어가야지, **_템플릿 _문자열_ 자체_** 가 되어서는 안 된다.** 한 번이라도 사용자 입력이 _템플릿 문자열_ 의 일부가 되면, 거기서부터 SSTI 위험이 시작됩니다.

### 4-3. 약한 비밀키 — _공개되어선 안 되는 값이 공개되었을 때_

Flask `secret_key` 가 _깃 저장소_ 에 그대로 박혀 있는 사례는 _보안 사고의 단골_ 입니다. 운영 권장:

- _**시크릿 매니저**_ (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault) + 런타임 주입
- 못해도 _환경 변수_ (`os.environ['SECRET_KEY']`) + `.env` 파일은 `.gitignore`
- _데모/예제 코드_ 라도 _"실서비스 전에 반드시 교체"_ 주석을 강하게 박는다

### 4-4. IDOR / 권한 검증 누락 — _"인증 ≠ 권한"_

`if session.get('uid')` 같은 _**로그인 여부**_ 검사는 _**인증(authentication)**_ 입니다. _이 사용자가 이 자원에 대해 무엇을 해도 되는가_ 는 _**권한(authorization)**_ 의 영역이며, _별도로_ 검증해야 합니다. 보통 이런 모양으로 들어갑니다:

```python
resource = db.get_resource(id)
if resource.owner_id != current_user.id and not current_user.is_admin:
    abort(403)
```

_좀 더 큰 패턴_: 모든 _자원 조회/수정_ 핸들러를 _데코레이터_ 로 통합 검증, 또는 _ORM 단계에서_ `Resource.objects.filter(id=id, owner=current_user)` 로 _쿼리 자체를 사용자 범위로 제한_. 이렇게 하면 _개발자가 한 줄을 잊어서_ IDOR 가 생기는 사고가 줄어듭니다.

## 5. 정리 — 입문자가 가져갈 교훈

- **SLA가 있는 패치 챌린지의 첫째 원칙: _**최소 침습**_**. _공격 표면만_ 잘라낸다. _**리팩토링/스타일 변경 금지**_. 한 줄을 더 건드릴수록 SLA가 깨진다.
- **취약점 클래스마다 _**한 줄짜리 표준 패치**_** 가 있다. 머릿속에 정답 형태를 갖춰두면 5분이면 끝난다. (SQLi → `?`+`(...)`, SSTI → Jinja 변수, 비밀키 → `os.urandom`, IDOR → owner 체크)
- **두 단계 _문자열 처리_ 결합은 SSTI 의 강한 시그널**. `format()` → `render_template_string`, `markdown.render()` → `Jinja`, `gettext()` → `f-string` 등이 한 줄 안에 있으면 _자동 적신호_.
- **`session.get('uid')` 만 보고 안심하지 말 것**. _"이 사용자가 이 자원을 _진짜로_ 만질 수 있는가"_ 는 _별도 조건_ 으로 검증.
- **제출 cooldown(30초) 이 _실험 비용_ 을 키운다**. 머리로 5분 더 검토하고 _한 번에_ 통과시키는 게 _30초씩 5번_ 돌리는 것보다 빠르다.

이 문제는 _100 라인_ 안에 _Flask 웹 보안의 표준 분류 4종 (Injection / Auth / Crypto Misuse / Access Control)_ 을 모두 모아둔, _**보안 코드 리뷰의 미니 표준 시험**_ 같은 챌린지입니다. 다음에 다른 패치 챌린지(예: PATCH-2, PATCH-3)를 만나도 _같은 체크리스트_ 를 적용하면 됩니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. `curl -d "app.py=..."` 가 500을 뱉었다 — `--data-urlencode` 였어야

첫 제출에서 `-d "app.py=$B64"` 형태로 보냈더니 _500 Internal Server Error_ 가 돌아왔습니다. base64 문자열 안의 `+` 가 _form 의 공백_ 으로 해석되면서 base64 디코드가 실패한 것이 원인. `--data-urlencode "app.py=$B64"` 로 바꾸자 정상 동작.

**회고**: base64 / 임의 바이너리 데이터를 form 으로 보낼 때는 _항상_ `--data-urlencode` 또는 `--data-binary --header "Content-Type: ..."` 을 쓴다. _첫 시도부터_ `-d` 의 _공백 변환 동작_ 을 의심하지 않으면 디버깅에서 5분 잃는다.

### 실패 2. _권한 검증_ 패치를 _안 해도 통과하는지_ 확신이 없었다

`memoUpdate` 의 권한 검증 누락을 _SLA가 강제하는지_ 알기 어려웠습니다. 만약 SLA가 _다른 사용자의 메모도 수정 가능해야 한다_ 라고 검증한다면 (예: _legacy 호환성 테스트_) 권한 체크를 넣자 마자 SLA가 깨질 수 있었습니다. 결과적으로 _권한 체크를 넣어도 ALL PASS_ 로 나왔지만, _넣어야 하나 말아야 하나_ 의 _**경계 판단**_ 이 첫 제출의 가장 큰 망설임이었습니다.

**회고**: 패치 챌린지에서 _가장 안전한 순서_ 는 _**(a) 명백한 보안 결함만 _최소_ 패치 → 제출 → ALL PASS 면 종료, FAIL 이면 (b)에서 단계적 강화**_. 본 풀이는 _**한 번에 ALL PASS를 노린**_ 풀어서 _모든 결함을 한 번에_ 패치했고, 다행히 통과했지만 _안전한 전략_ 은 _최소부터_ 입니다. 다음 패치 챌린지에서는 _**1차: SQLi + SSTI 만 / 2차: secret_key 추가 / 3차: 권한 체크 추가**_ 의 _점진적 제출_ 을 시도해 _**SLA가 무엇을 검사하는지 역으로 가늠**_ 해 보겠습니다.

### 실패 3. `query_db(...)[0]` 의 IndexError 위험을 처음엔 _고치고 싶어_ 했다

원본 코드는 `query_db("SELECT ... where idx=...")[0]` 으로, _idx에 해당하는 행이 없으면_ `IndexError` 가 나서 _500 응답_ 이 됩니다. _보안 패치_ 의 _깨끗함_ 을 추구하다 보면 _**`if not ret: return ...`**_ 같은 _가드 절_ 을 추가하고 싶어집니다. 그러나 _그 변경_ 이 _SLA의 어떤 검증_ 을 깨뜨릴지 모르므로, _기능 동작_ 을 그대로 유지하는 게 _덜 위험_ 합니다. 결국 _가드 절을 추가하지 않은 채_ 통과.

**회고**: _**보안과 무관한 robust 화는 자제**_. _패치 챌린지는 _"보안 결함만 잘라내기"_ 가 목표_, 아닌 _"잘 다듬어진 코드 만들기"_ 가 아니다. _**원본의 _정상 경로_ 동작을 비트로 일치시키는 것**_ 이 SLA 통과의 첫 번째 비결.

이 세 가지가 다음 패치 챌린지에서 시간을 가장 많이 줄여줄 학습 포인트입니다.
