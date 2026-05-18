---
layout: post
title:  "Dreamhack 워게임 Where-is-localhost 풀이 — IPv4-Mapped IPv6 (`::ffff:127.0.0.1`) 로 _IPv4 차단_ 우회"
date:   2026-05-19 02:30:00 +0900
categories: security web wargame dreamhack writeup
---

## 들어가며

이번 글에서는 Dreamhack 웹해킹 워게임 **Where-is-localhost** (Unrated, 풀이자 440+ 명) 문제를 정보보안 입문자의 시점에서 풀어봅니다. 제목을 풀이의 첫 단서로 봅시다 — _**"localhost 는 어디에 있는가?"**_. 정답은 _**"여러 곳에**_ 있다, 그리고 그 사실을 _필터가 다 알지 못한다_" 입니다.

문제 설명은 단 두 줄.

> localhost... 과연 어떻게 접근할까?
> Edit (10/22/2024 7:49): fix error in code

핵심은 _localhost 접근_ — 즉, 서버가 _스스로에게_ HTTP 요청을 보내게 만들면 플래그를 주는 _SSRF (Server-Side Request Forgery) 라이트 버전_ 입니다. 다만 _"필터가 IPv4 를 정직하게 막아두었으니, 어떻게 우회할 것인가"_ 가 진짜 질문입니다.

> Spoiler: 풀이는 **`vulntest=::ffff:127.0.0.1`** 이라는 한 줄 페이로드입니다. 이 _열다섯 글자_ 안에 _**리눅스 듀얼 스택 소켓의 IPv4-Mapped IPv6 변환**_ 이라는 _운영체제 수준의 동작_ 이 숨어 있습니다.

## 1. 문제 파일 들여다보기

```
Where-is-localhost/
├── Dockerfile        ← python:3.13-alpine, flask 만 설치
└── main.py           ← 45 줄, 단일 파일
```

`main.py` 의 본문은 압축하면 다음과 같습니다.

```python
@app.route('/vuln', methods=['POST'])
def vuln():
    name = request.form.get('vulntest')
    try:
        address = ipaddress.ip_address(name)
        if address.version == 4:                                # (a) v4 차단
            return "no..."
        url = urllib.parse.urlparse(f"http://[{address.exploded}]:5000/localonly")
        if url.netloc != f'[{address.exploded}]:5000':          # (b) 자기 검증
            return "no..."
        req = urllib.request.Request(url.geturl())
        return urllib.request.urlopen(req).read().decode('utf-8')   # (c) 자기 호출
    except ValueError:
        return "no..."
    except urllib.error.URLError:
        return "connection refused"

@app.route('/localonly', methods=['GET'])
def localonly():
    addr = ipaddress.ip_address(request.remote_addr)
    if addr.is_loopback and addr.version == 4:                  # (d) v4 루프백만 허용
        return flag
    else:
        return 'not loopback'
```

_즉시 표시해 둘_ 네 줄.

- **(a)** 입력 `name` 을 `ipaddress.ip_address()` 로 _파싱_. _**파싱 결과가 IPv4 면 거절**_. → 사용자는 _**IPv6 만 보내야 한다**_.
- **(b)** `address.exploded` 로 _완전 형식_ IPv6 (`0000:0000:...`) 을 만들어 URL 을 조립. `urlparse(...).netloc` 이 _자기 조립한 그 문자열_ 과 정확히 같은지 검증. _**구문적으로 IPv6 인지를 두 번 확인**_ 하는 셈.
- **(c)** 만든 URL 에 `urllib.request.urlopen` 으로 _**서버 스스로**_ HTTP GET. → 서버 → 서버 자체로의 요청이 발생.
- **(d)** `/localonly` 는 _요청자의 IP가 **IPv4** 이고 **loopback** 일 때만_ 플래그를 돌려준다.

즉, _공격자_ 의 의무는 다음 _세 조건을 동시에_ 만족시키는 입력을 찾는 것:

1. **(a) 통과** : `ipaddress.ip_address(name)` 결과가 _**IPv6**_.
2. **(c) 실제 연결** : urllib 의 `connect()` 가 _**127.0.0.1 (IPv4 루프백)**_ 으로 도달.
3. **(d) 통과** : Flask 가 보는 `request.remote_addr` 가 _**IPv4 루프백 문자열**_ ("127.0.0.1").

_"입력은 IPv6 인데, 실제 연결은 IPv4 루프백"_ — 이게 가능할까? 한 줄 더 들어가면 가능합니다. **리눅스의 듀얼 스택 소켓** 이 _IPv4-Mapped IPv6 주소_ 를 _IPv4 트래픽_ 으로 변환합니다.

## 2. 핵심 개념 — _IPv4-Mapped IPv6 Address_

IPv6 명세 (RFC 4291) 는 _**IPv4 주소를 IPv6 표기로 표현하는 특별한 prefix**_ `::ffff:0:0/96` 를 정의합니다. 형태는

```
::ffff:a.b.c.d        # 가독성 표기
::ffff:0:7F00:1       # 16비트 그룹 표기
0000:0000:0000:0000:0000:ffff:7f00:0001   # 완전 형식 (exploded)
```

이런 주소들은 _**구문적으로는 IPv6 (128비트)**_ 이지만, _**의미적으로는 IPv4 주소 a.b.c.d 를 가리킵니다**_. 그리고 **리눅스 듀얼 스택 소켓** 은 _이 형식의 주소로 `connect()`_ 가 일어나면, 내부적으로 _**진짜 IPv4 패킷**_ 을 보냅니다. 받는 쪽이 _IPv4 만 listen 하는 서버_(예: `app.run('0.0.0.0', 5000)`) 라도 _**문제 없이 연결**_ 됩니다.

이 _**구문(IPv6) - 의미(IPv4) 갭**_ 이 정확히 본 챌린지의 우회 통로입니다.

각 조건을 다시 점검해 봅시다.

- **(a) `ipaddress.ip_address('::ffff:127.0.0.1').version`** : _**6**_. → _v4 차단 통과_.
- **(b) `address.exploded`** : `'0000:0000:0000:0000:0000:ffff:7f00:0001'`. _urlparse_ 가 만든 netloc 도 정확히 `[0000:0000:0000:0000:0000:ffff:7f00:0001]:5000`. → _자기 검증 통과_.
- **(c) urllib `connect()`** : 듀얼 스택 소켓이 _IPv4 패킷_ 으로 변환. 127.0.0.1:5000 의 _Flask 자기 자신_ 에 도달.
- **(d) `request.remote_addr`** : Werkzeug 는 _실제로 TCP 위에서 받은 IPv4_ 를 보므로 _**`"127.0.0.1"`**_. `ipaddress.ip_address("127.0.0.1").version == 4` _AND_ `.is_loopback == True`. → _플래그 반환_.

조건 4개를 _한 줄_ 로 다 통과. 풀이 끝.

## 3. 풀이 — 한 줄 `curl`

```bash
URL=http://host8.dreamhack.games:22267
curl -s -X POST -d "vulntest=::ffff:127.0.0.1" "$URL/vuln"
# → DH{L0C4L_En0ugh_IPv4_mapped_v6-a44dc189}
```

대조군으로 _순수 IPv6 loopback_ 도 시도해 봅니다.

```bash
curl -s -X POST -d "vulntest=::1" "$URL/vuln"
# → connection refused
```

왜 _`::1`_ 은 안 되나? 두 가지 이유 중 하나입니다.

1. Flask `app.run('0.0.0.0', 5000)` 은 _**IPv4 만 listen**_. `::1` 로 가는 _IPv6 트래픽_ 을 받아주는 listener 가 없음 → 즉시 `ECONNREFUSED`.
2. 설사 listen 했더라도 _도착 IP_ 가 _IPv6 loopback (`::1`)_ 이라 `.version == 6` → `/localonly` 가 _"not loopback"_ 반환.

즉 _**v6 loopback 은 _구문도 의미도 모두 v6_**_ 라서 (d) 가 막습니다. _**IPv4-Mapped IPv6 만 _구문은 v6, 의미는 v4_**_ 라서 (a) 도 (d) 도 모두 우회됩니다.

## 4. 왜 이런 우회가 _**구조적으로 흔한가**_

이 문제는 _IP 주소 처리 시 가장 잘 알려진 SSRF 우회 패턴_ 의 미니어처입니다. 같은 결의 우회를 _실무_ 에서 보는 흔한 모양:

### 4-1. _Localhost 차단을 우회하는 표준 가젯들_

비슷한 케이스에서 _**공격자가 머리에 갖춰두는**_ 우회 표 (대표 7가지):

| 의도한 표적 | 우회 입력 | 왜 통과되나 |
|---|---|---|
| `127.0.0.1` | `127.0.0.0/8` 안의 임의 주소 (`127.1.2.3`) | 단순 문자열 매칭만 막은 경우 |
| `127.0.0.1` | `0.0.0.0` | 리눅스에서 자기 자신을 의미할 때 있음 |
| `127.0.0.1` | `127.1` (단축) | inet_aton 의 _옥텟 축약_ 지원 |
| `127.0.0.1` | `2130706433` (= `0x7F000001` 십진) | inet_aton 의 _정수_ 입력 지원 |
| `127.0.0.1` | `localhost.attacker.com` (A record → 127.0.0.1) | _**DNS rebinding**_ |
| `127.0.0.1` | `::ffff:127.0.0.1` (IPv4-Mapped IPv6) | **본 문제의 정답** |
| `127.0.0.1` | `[::]` (IPv6 unspecified) | IPv4 의 `0.0.0.0` 와 같은 의미 |

본 문제는 _**여섯 번째 행**_ 의 _정석적인 케이스_ 입니다. 다른 행들도 _서로 다른 코드 패턴_ 에 대응하므로, _체크리스트 형태로_ 머릿속에 두면 SSRF 류 문제에서 _첫 30초_ 가 빨라집니다.

### 4-2. _CVE 사례_ — 같은 결의 실 사고

- **Kubernetes API server SSRF (CVE-2020-8557)** : pod 의 _exec_ 엔드포인트 우회에 _localhost 변형_ 이 동원.
- **GitLab CI runner SSRF (CVE-2021-22264)** : DNS rebinding 으로 _내부 메타데이터_ 접근.
- **Cloud metadata exfil** : AWS `169.254.169.254` 에 _localhost 우회 + 헤더 조작_ 으로 IAM 자격 탈취 (Capital One 사고). 다양한 형태의 _IP 표기 우회_ 가 가장 흔한 페이로드.

### 4-3. _왜 표준 라이브러리는 이걸 "수정" 하지 않는가_

`ipaddress` 모듈은 _**RFC 표준대로 정확하게**_ 동작합니다. `::ffff:127.0.0.1` 은 _RFC 4291 정의상 IPv6 주소가 맞습니다_. 따라서 `.version` 이 6 인 것은 _버그가 아니라 명세_. 마찬가지로 _듀얼 스택 소켓의 IPv4-Mapped 변환_ 도 _RFC 3493 정의상 정상 동작_.

문제는 _개발자가 _구문 분류_ 와 _의미 분류_ 를 혼동_ 하는 것입니다. _**`.version == 4` 가 `is_actually_ipv4_destination == True` 와 동치라고 착각**_ 하는 순간 게이트가 깨집니다.

## 5. _안전한_ 패치는 어떻게 짜야 할까

본 문제는 패치 챌린지가 아니지만, _**같은 코드를 보안 리뷰한다고 생각하면**_ 다음과 같이 고칠 수 있습니다.

```python
from ipaddress import ip_address, IPv6Address

def is_safe(host_str: str) -> bool:
    a = ip_address(host_str)
    # IPv4-Mapped, IPv4-Compatible 모두 IPv4 주소로 _정규화_ 한 뒤 검사
    if isinstance(a, IPv6Address) and a.ipv4_mapped is not None:
        a = a.ipv4_mapped
    if a.is_loopback or a.is_link_local or a.is_private or a.is_reserved:
        return False
    return True
```

핵심은 _**`.ipv4_mapped` 속성을 명시적으로 정규화**_ 하는 것. 표준 라이브러리가 친절하게 _IPv4-Mapped IPv6 → IPv4_ 변환을 _바로 노출_ 합니다. 단순 `.version` 검사 대신 _정규화 → 분류_ 의 두 단계가 정답.

또 다른 방어선은 _**connect() 시점 검증**_. `urllib` 의 _custom HTTPHandler_ 로 _소켓 연결 직전에 실제 도착 IP_ 를 다시 검사하면, _DNS rebinding_ 까지 함께 차단할 수 있습니다. 이는 _**TOCTOU (Time-Of-Check Time-Of-Use)**_ 문제로 정리됩니다 — 검사 시점과 사용 시점이 다르면 우회가 생긴다는 _고전적 안티 패턴_.

## 6. 정리 — 입문자가 가져갈 교훈

- **_구문(syntax)_ 분류와 _의미(semantics)_ 분류는 다른 차원이다.** `ipaddress.ip_address(x).version == 4` 는 _"`x` 가 IPv4 구문이냐"_ 만 답한다. _"이 주소가 IPv4 목적지로 가느냐"_ 는 _별도 분류_.
- **`::ffff:a.b.c.d` 는 _IPv6 가 입은 IPv4 의 옷_**. 듀얼 스택 OS 가 자동으로 _IPv4 패킷_ 으로 변환. 이 _구문-의미 갭_ 이 SSRF 우회의 흔한 통로.
- **SSRF 차단 코드를 짤 때는 _**`ipaddress.ip_address(...).ipv4_mapped` 정규화**_** 를 반드시 한 단계 거친 뒤 분류하라. 그리고 가능하면 _socket connect 시점에서_ 다시 검증하라 (DNS rebinding 까지 막힌다).
- **localhost 우회 가젯 7종** (위 표) 을 _체크리스트_ 로 두면 _첫 30초_ 에 정답을 가져온다.
- **`address.exploded` 같은 _완전 형식_ 검증** 은 _IPv6 구문 변형_(축약, scoped) 을 막을 뿐, _IPv4-Mapped 의미 우회_ 는 막지 못한다.

이 문제는 _**`ipaddress` 모듈을 _진지하게_ 다뤄 본 적 없는 개발자가 본의 아니게 만드는 가장 흔한 결함**_ 의 _실험실 표본_ 입니다. _RFC + 듀얼 스택 동작_ 을 알고 있다면 _코드를 읽는 즉시_ 정답이 보이고, 모른다면 _아무리 코드를 봐도 안 보이는_ — _지식 의존형_ 문제의 모범입니다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. _`::1` 부터 시도_ 한 것은 자연스러웠으나 답이 아니었다

처음 _localhost_ 라는 단어를 보고 _IPv6 loopback `::1`_ 을 먼저 시도했습니다. 결과는 _connection refused_. _**Flask가 IPv6 를 listen 하지 않는다**_ 는 점, 그리고 _**도착 IP 가 v6 인 순간 (d) 가 막는다**_ 는 점 두 가지 모두를 만족시키지 못합니다.

**회고**: _"localhost"_ 라는 단어는 _v4 _와_ v6_ 두 곳을 동시에 가리킨다. _**v6 loopback ≠ v4 loopback**_. 본 문제처럼 _(d) 가 v4 만 인정_ 하는 형태라면 _순수 v6 loopback 은 정답이 될 수 없다_. 다음부터는 _목표 조건 (d)_ 을 먼저 만족시키는 _도착 측 IP_ 를 _**역추적**_ 한 뒤 _입력 구문_ 을 정한다 — _**역방향 사고**_ 가 _순방향 시도_ 보다 빠르다.

### 실패 2. `address.exploded` 검증 (b) 가 _무력한 검증_ 임을 알아채는 데 잠시 걸렸다

`url.netloc != f'[{address.exploded}]:5000'` 검사는 _**같은 변수**_ 두 곳을 비교하므로 _구조적으로 거의 항상 통과_ 합니다. _진짜 의도_ 는 _다른 IPv6 표기(축약, scoped, 임베디드 v4) 가 들어왔을 때 정규화 일관성을 확인_ 하는 것이지만, _이 정규화 자체가 IPv4-Mapped 변환을 막지 못합니다_. 이 사실을 알기 전에 잠시 _"이 검증이 정답을 막는 게 아닐까"_ 라는 옆길로 새었습니다.

**회고**: _**"비교 양쪽이 같은 변수면 그 검사는 _장식_ 일 가능성이 높다"**_. 이 경우 _검사가 잡으려는 클래스가 실제로 막히는가_ 를 _즉시_ 확인하면 1분을 아낀다.

### 실패 3. 처음엔 _**DNS rebinding**_ 부터 떠올렸다

_"localhost 우회"_ 라는 키워드만 듣고 _**DNS rebinding (`attacker.com → 127.0.0.1`)**_ 부터 떠올렸습니다. 그러나 본 문제는 _**입력을 `ipaddress.ip_address()`로 _즉시 파싱_**_ 하므로 _호스트네임은 들어가지 않습니다_. DNS rebinding은 _호스트네임이 허용되는_ 코드에서만 동작.

**회고**: _**우회 가젯_ 은 _코드의 받아들이는 입력 형식_ 에 따라 _쓸 수 있는 것이 다르다_**. 첫 30초에 _"이 코드는 어떤 입력 형식까지 허용하는가"_ 를 먼저 정의한 뒤, _그 안에서_ 우회 가젯을 매칭한다. 가젯 목록을 _**무작위로 시도**_ 하는 것이 가장 비효율적.

이 세 가지가 다음 SSRF/IP 분류 우회 문제에서 _첫 30초 의사결정_ 을 빠르게 해줄 학습 포인트입니다.
