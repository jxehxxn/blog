---
layout: post
title:  "Dreamhack 워게임 Ctrl-C 풀이 — 드래그·우클릭·F12 막아도 curl 한 줄에 끝나는 이유"
date:   2026-05-16 09:00:00 +0900
categories: security web wargame dreamhack writeup javascript
---

## 들어가며

Dreamhack 웹해킹 워게임 **Ctrl-C** (난이도 Bronze 4) 풀이입니다. 사이버보안 입문 학생을 위한 **"클라이언트 UX 제약 ≠ 보안"** 명제를 가장 짧고 명확하게 보여주는 교과서 문제입니다.

문제 설명:

> Copy it if you can! 플래그의 형식은 `WaRP{...}` 입니다.

스포일러로 결론: 페이지 상에서는 텍스트 드래그/우클릭/F12 가 모두 막혀 있어 보이지만, 그 모든 차단은 **JavaScript 안에서만** 작동합니다. JS 가 실행되지 않는 `curl` 한 줄로 페이지를 받아오면 원본 텍스트가 그대로 들어옵니다.

```bash
# 1) 페이지 받아 텍스트 추출 → 2) 제공된 스크립트에 그대로 입력
curl -s $SERVER/ | python3 -c "import sys, re; print(re.search(r'<p>([^<]+)</p>', sys.stdin.read()).group(1))" \
  | python3 Ctrl_C.py
# → b'WaRP{1_w4n7_5h0r7cu7_k3y}'
```

> Note: 본 글은 사이버보안 입문 학습용 PoC. 실제 운영 서비스의 카피 방지 UI 를 우회해 콘텐츠를 무단 복제하는 행위와 구분.

## 1. 자료 구성

ZIP 에는 두 파일만 있다.

```
Ctrl-C/
├── Dockerfile
└── Ctrl_C.py        ← 디코딩 스크립트
```

`Dockerfile` 은 `app.py`, `text.txt`, `templates/index.html` 을 참조하지만 ZIP 에는 그것들이 없다 — 즉, **클라이언트가 보는 페이지는 라이브 서버에 직접 가서 받아와야** 한다.

### 1-1. `Ctrl_C.py` — 디코딩 스크립트

```python
import hashlib, base64

def bytes_to_long(a): return int.from_bytes(a, byteorder='big')
def long_to_bytes(a): return a.to_bytes((a.bit_length()+7)//8, byteorder='big')

print(long_to_bytes(
    548488142063681088110499188198346596132432266189304030893626
    ^ bytes_to_long(base64.b64encode(hashlib.sha256(input().rstrip().encode('utf-8')).digest())[13:36])
))
```

한 줄로 정리하면:

```
output = MAGIC XOR (base64(sha256(input))[13:36] 의 23바이트를 정수로 본 값)
       을 다시 bytes 로
```

즉 **input 으로 무엇을 주면 output 이 `WaRP{...}` 가 되는가?** 를 풀어야 한다. 임의로 해시 preimage 를 만들 순 없으니, **결정적 정답 input 이 서버 어딘가에 있어야** 한다.

### 1-2. 라이브 페이지 - 정답 input

`curl $SERVER/` 으로 받아본 HTML 의 `<p>...</p>` 안에 다음과 같이 2000자짜리 base64-비스무리한 긴 문자열이 들어 있다.

```
XESBGo7MvS8ia39lltRaFeWj3ptSSC45fanWNbCV6GXmt6VefXn27Oo79jfrFMUQKcLeZ ...
... rOsvNrpQ9fmj75EC3HHA0ewPlpTBO6EFBg1SNbEtLfNFH7WO8KJz2JDekpN1fLYN63K1SQcYKlvlgXXpnlW6ykGZOgRz13JIMx1rJI63NwGSbi6QyI2aoQanwCJiWWbSLuElfP39ZT47ctewsumXcMmT
```

이 문자열을 `Ctrl_C.py` 의 stdin 으로 넘기면 출력이 `b'WaRP{...}'` 인 flag 가 나온다. 즉 **이 긴 문자열이 곧 정답 input**.

여기까지의 그림:

```
라이브 페이지 <p>긴 문자열</p>
       │ (Ctrl-C 로 복사해서)
       ▼
Ctrl_C.py 의 stdin
       │
       ▼
flag 출력
```

## 2. 무엇이 "방해" 인가 — 페이지의 JS

```javascript
// 우클릭 방지
document.oncontextmenu = function() { return false; }

// 드래그 / 텍스트 선택 방지
document.onselectstart = new Function("return false");
document.onmousedown = disableselect;

// F12 / Ctrl+Shift+I / Ctrl+Shift+J 차단
document.addEventListener("keydown", function(e) {
    if (e.keyCode == 123) e.preventDefault();
    if (e.ctrlKey && e.shiftKey && (e.keyCode == 73 || e.keyCode == 74))
        e.preventDefault();
});
```

브라우저에서 사용자가 마우스로 선택해서 Ctrl-C 하는 경로를 막는 흔한 코드.

### 2-1. 이 모든 방어가 "한 줄" 로 무력화되는 이유

- **JS 는 클라이언트에서만 실행** 된다. HTTP 요청과 응답에는 JS 의 어떤 차단도 끼어들 수 없다.
- `curl` / `wget` / Python `requests` 는 JS 를 평가하지 않는다. **응답 본문은 항상 원본 HTML 그대로** 받아진다.
- 브라우저로 보더라도, JS 를 비활성화하거나 `view-source:` 스킴으로 보면 같은 결과.

## 3. 풀이 (한 줄)

```bash
SERVER=http://<host>:<port>

# 1) 페이지에서 <p>...</p> 텍스트만 추출
TEXT=$(curl -s "$SERVER/" | python3 -c \
  "import sys, re; print(re.search(r'<p>([^<]+)</p>', sys.stdin.read()).group(1))")

# 2) Ctrl_C.py 에 stdin 으로 그대로 넣기
echo "$TEXT" | python3 Ctrl_C.py
# → b'WaRP{1_w4n7_5h0r7cu7_k3y}'
```

또는 파이프 한 줄로:

```bash
curl -s "$SERVER/" \
 | python3 -c "import sys, re; print(re.search(r'<p>([^<]+)</p>', sys.stdin.read()).group(1))" \
 | python3 Ctrl_C.py
```

flag: `WaRP{1_w4n7_5h0r7cu7_k3y}` ✓

## 4. 취약점 해설 — Client-Side "Security" vs. Server-Side Reality

이 클래스의 본질은 입문서 1쪽에 나올 만큼 단순.

> 클라이언트가 거는 모든 제약(JS 차단, UI 비활성화, CSS 숨김 처리 등) 은 **사용자 경험상의 제약** 이지 보안 제약이 아니다. 서버가 같은 제약을 강제하지 않는다면, HTTP 호출 한 줄로 우회된다.

### 4-1. 같은 패턴의 실무 사례

이 문제가 가르치는 패턴은 단순해 보이지만, 실제로는 너무 자주 보안 사고로 이어진다.

- 결제 페이지의 **JavaScript-only 금액 검증** (`if (price < 0) ...`). 클라이언트가 금액 0원을 직접 POST → 결제 통과.
- 가입 페이지의 **클라이언트 측 비밀번호 정책** (`if (password.length < 8) reject`). 서버 미검증 → 1자리 비밀번호 가입.
- 권한 확인 후 메뉴를 **CSS 로 숨김** (`.admin-only { display: none; }`). 사용자가 직접 admin URL 입력 → 그대로 접근.
- 다운로드 페이지의 **다운로드 횟수 제한 JS 카운터**. 사용자가 페이지 새로고침 또는 직접 다운로드 URL 호출 → 무한 다운로드.
- 게임 클라이언트의 **쿨다운 / HP 검증** 을 클라이언트에서만 함. 봇이 쿨다운 무시하고 RPC 호출 → cheating.

### 4-2. 위험성

- 사용자의 우회 능력은 매우 일반화된다 — F12 / 브라우저 확장 / curl 한 줄 / 모바일 앱 디컴파일 등.
- "JS 가 모든 사용자에게 정상 실행될 것이다" 라는 가정은 **AdBlock, NoScript, 봇, 자동화 도구, API 클라이언트** 만 떠올려도 즉시 깨진다.

### 4-3. 올바른 방어

1. **모든 보안 결정은 서버에서.** 클라이언트의 UI 제약은 "친절한 안내" 일 뿐 의무가 아니다.

2. **민감 데이터는 클라이언트에 보내지 마라.** flag 가 페이지 본문에 박혀 있는 한, 어떤 JS 로도 복사를 막을 수 없다. 보호하고 싶은 데이터는 애초에 클라이언트로 내려보내지 말 것.

3. **DRM 류 콘텐츠 보호가 필요하다면 전용 솔루션** (Widevine 같은 EME 기반). 그래도 100% 막지 못한다 — 화면을 그냥 사진 찍을 수 있으니까. "공격자가 화면을 볼 수 있다면 그 콘텐츠는 결국 유출된다" 는 절대적 진실.

4. **API 에 인증/권한 검증을 항상 별도로**. 메뉴 숨김으로 권한 보호하지 말 것.

## 5. 정리 — 입문자가 가져갈 교훈

- **클라이언트에서 거는 모든 제약은 장식.** UI 의 timer, captcha, modal, hidden field, disabled button, drag-prevention, right-click-disable, F12-block — 모두 HTTP 요청 한 줄로 우회.
- 사용자에게 데이터를 보낸 시점에서 **그 데이터는 사용자 통제 하** 다. "복사하지 못하게 한다" 는 본질적으로 불가능.
- F12 차단 같은 JS 는 **사용성을 해치면서도 보안에 도움이 0**. 차라리 안 거는 게 낫다 (사용자 분노 + UX 손상).
- 같은 페이지를 브라우저가 아니라 `curl` 로 받았을 때 무엇이 보이는지 항상 점검. 보호 의도 가졌던 어떤 것도 raw HTML 에 박혀 있으면 보호 실패.

같은 카테고리의 다음 단계로 **Test Your Luck**(client-only timer), **Click me!**(client-only escape), 그리고 **이 글의 Ctrl-C**(client-only copy block) 가 정확히 같은 클래스의 3종 세트. 사이버보안 입문자라면 세 개를 묶어서 풀면 "클라이언트의 모든 것은 위조 가능" 이라는 명제가 손에 박힌다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. ZIP 에 app.py 가 없어서 한 번 멈춤

ZIP 안에는 `Ctrl_C.py` (디코딩 스크립트) 와 `Dockerfile` 만 있고, Dockerfile 이 참조하는 `app.py`, `text.txt`, `templates/index.html` 은 빠져 있다. 처음에 "출제자가 빼먹은 건가?" 하고 잠깐 멈췄다.

곧 이게 의도였다는 걸 인지: **라이브 페이지에 가야 input 텍스트를 얻을 수 있다**. ZIP 의 `Ctrl_C.py` 는 그저 "이걸 어떻게 디코딩하는가" 만 알려주는 도구.

**회고**: ZIP 의 Dockerfile 이 참조하는 파일이 ZIP 에 없으면 의심하기 전에 한 번 더 생각하자. 출제자가 일부러 "**라이브 페이지에서 가져와야 한다**" 라는 메시지를 주는 경우가 많다. "이 정보는 어디서 가져와야 하는가?" 를 그림으로 그려보자.

### 실패 2. `<p>` 추출 정규식 한 번 잘못

처음에 `re.search(r'<p>(.*)</p>', html)` 로 잡았는데, HTML 안에 다른 `<p>` 가 있다면 greedy 매칭으로 큰 범위를 잡을 수 있다. 그래서 `<p>([^<]+)</p>` 로 변경 — `<` 이전까지만 매칭.

이번 페이지엔 `<p>` 가 하나뿐이라 영향 없지만, 정규식 습관을 들이자.

**회고**: HTML 에서 텍스트 추출은 **lxml/BeautifulSoup** 가 정공법. 일회용 PoC 에서는 정규식이라도 **`[^<]+` 처럼 closing tag 까지 안 가는 패턴** 이 안전.

### 실패 3. 처음에 `Ctrl_C.py` 코드를 "preimage 공격" 으로 잘못 읽음

처음에 코드를 보고 "주어진 출력을 만드는 input 을 찾아야 하나? SHA256 preimage 는 불가능한데..." 라고 잠시 헤맸다. 다시 보니 **반대 방향** — input 을 주면 output 이 나오는 형태이고, 정답 input 이 라이브 페이지에 있다. "preimage 찾기" 는 잘못된 모델.

**회고**: 코드의 입출력 방향을 정확히 읽자. `input() → output` 인지, `output 으로부터 input 을 복원` 인지. SHA256 같은 비가역 함수가 보인다고 무조건 preimage 공격이 아니다. 정답 input 이 다른 곳(라이브 페이지, 로그, README, etc.) 에 있는지 먼저 확인.

### 실패 4. `echo "$TEXT"` 가 줄바꿈을 추가하는 점

`echo "$TEXT" | python3 Ctrl_C.py` 했더니 처음에 결과가 살짝 이상해 보였다. `echo` 는 끝에 줄바꿈을 추가하지만, `Ctrl_C.py` 는 `input().rstrip()` 으로 trailing whitespace 를 제거하므로 영향 없음. 결국 정상 작동.

**회고**: shell pipe 에서 텍스트 흘릴 때 trailing newline / spaces 가 결과를 바꾸는지 항상 의식하자. `rstrip()`/`strip()` 이 있는지, 또는 `echo -n` 을 쓸지 한 번 더 점검.
