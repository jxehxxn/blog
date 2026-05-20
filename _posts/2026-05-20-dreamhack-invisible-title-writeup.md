---
layout: post
title:  "Dreamhack 워게임 (제목 보이지 않음 / id 2182) 풀이 — Unicode 비표시 문자 U+200E 만 끼워넣은 OSINT 워밍업"
date:   2026-05-20 09:30:00 +0900
categories: security web wargame dreamhack writeup unicode bidi forensics
---

## 들어가며

Dreamhack 웹해킹 워게임 **(제목 비표시, id 2182, 풀이자 388명)** 풀이입니다. 제목 자체가 zero-width / bidi 같은 비표시 유니코드 문자로 채워져 있어서 사이트에서도 빈칸으로 보이는 장난스러운 출제예요. 본질은 "사람 눈에 보이는 것 ≠ 컴퓨터가 읽는 것" 한 줄.

문제 설명도 같은 비표시 문자 한 글자:

> (보이지 않음)

스포일러 결론:

```bash
unzip source.zip          # README.txt 한 파일
cat README.txt            # 사람 눈엔 'DH{Do_You_Believe_...}' 같이 보이는데 사이사이 깨진 느낌
python3 -c '
import unicodedata
data = open("README.txt").read()
print("".join(c for c in data if not unicodedata.category(c).startswith("Cf")))
'
# → DH{Do_You_Believe_What_you_See_on_the_web?}
```

플래그 본문 그대로 — "웹에서 보이는 것을 그대로 믿지 말아라".

## 1. 자료 파악

ZIP 안에는 `README.txt` 한 파일. 16진 덤프를 떠보면:

```
e2 80 8e | e2 80 8e | e2 80 8e | 44 ('D') | e2 80 8e | 48 ('H') ...
```

`e2 80 8e` = UTF-8 인코딩의 **U+200E (LEFT-TO-RIGHT MARK)**. 양방향 텍스트(bidi) 의 방향을 LTR 로 강제하는 보이지 않는 제어 문자입니다. 이걸 각 글자 사이에 빼곡히 끼워넣은 게 README 의 전부.

```
‎‎‎D‎H{D‎o_‎‎Yo‎u‎_Be‎‎li‎‎e‎ve‎_‎W‎ha‎t‎_y‎o‎u‎_S‎e‎e‎_o‎n‎_t‎h‎e_‎w‎eb‎?‎}‎‎‎
   ↑ 위에 보이는 화살표 같은 흔적이 U+200E 자리
```

## 2. 풀이 — 비표시 문자 제거

가장 흔한 비표시(formatting) 문자들은 유니코드 카테고리 **`Cf`** (Format) 또는 **`Cc`** (Control) 에 속합니다. Python `unicodedata.category()` 로 한 줄 필터:

```python
import unicodedata
data = open("README.txt").read()
visible = "".join(c for c in data if not unicodedata.category(c).startswith("Cf"))
print(visible)
# DH{Do_You_Believe_What_you_See_on_the_web?}
```

또는 ASCII printable 만 남기는 더 거친 방법도 동일한 결과:

```python
ascii_only = "".join(c for c in data if ord(c) < 128 and c.isprintable())
```

서버 부팅도 필요 없는 5초 컷 문제. 출제자의 메시지가 **플래그 본문 자체에 들어가 있는 의도** 가 핵심: "사이트에서 보이는 글자 = 진짜 글자" 라고 가정하면 안 된다는 OSINT/시각 보안 워밍업.

## 3. 왜 이런 게 보안 이슈가 되는가

이게 단순 장난 같지만 실제 보안 사고와도 직결되는 패턴입니다.

### 3.1 비표시 문자가 만든 실제 사고들

- **호모글리프/IDN 스푸핑**: `g00gle.com`, `аpple.com` (키릴 а), `paypal[.]com` 의 P 를 ρ(rho) 로... 사람 눈엔 같아 보이는 도메인.
- **Trojan Source (CVE-2021-42574, 2021)**: 코드 안에 RLO/LRE 같은 bidi 제어 문자를 박아서 **컴파일러가 보는 코드** 와 **리뷰어가 보는 코드** 를 다르게 만드는 공격. C/Rust/JS 어디든 가능. GitHub 코드 리뷰가 비표시 문자를 강조해 보여주게 된 계기.
- **Slack/Discord 닉네임 위장**: ZWJ 를 닉네임 사이에 끼워서 같은 닉네임이지만 다른 사용자로 인식되게 함.
- **Phishing email 우회**: SpamAssassin 같은 룰 기반 필터에서 키워드 매칭을 ZWJ 한 글자로 깨뜨림.

### 3.2 방어 전략

- **표시 시점에 강조**: 코드/메시지를 표시할 때 비표시 문자를 시각적으로 드러내자 (`U+200E` 같은 마커 highlight). VS Code 와 GitHub 도 이걸 한다.
- **입력 단계 normalization**: 사용자 입력을 처리하기 전에 `unicodedata.normalize("NFKC", s)` 로 정규화 + 카테고리 `Cf`/`Cc` 문자 제거. 단 시도 자체를 막을지(거부) 아니면 제거(silent strip) 할지는 정책 결정.
- **도메인 비교는 punycode 로**: IDN 도메인은 punycode 변환 후 비교/표시. 브라우저들도 의심스러운 IDN 은 punycode 로 표시하는 정책을 둠.

## 4. 한 번 더 정리

### 4.1 취약점 카탈로그

| 단계 | 카테고리 | 원인 |
|---|---|---|
| 사람 눈으로만 읽음 | Visual Spoofing | 비표시/위장 문자 미인지 |
| 입력 normalization 부재 | Insufficient Input Validation (CWE-176) | 유니코드 정규화/필터 안 함 |

### 4.2 입문자가 챙겨가면 좋은 시각

- **사람이 본 것은 데이터의 한 표현일 뿐**이다. 진짜는 바이트.
- 의심스러운 문자열이 보이면 `hexdump`, `python3 -c "print(<str>.encode().hex())"`, `iconv`, `unicodedata.category()` 같은 도구로 **bytes-level 검사**부터.
- 유니코드 트릭은 OSINT, 사회공학, 코드 백도어, 도메인 스푸핑 등 광범위하게 쓰임. 한 번 익혀두면 평생 자산.

## 5. 시도했지만 실패한 것들 (회고)

이번엔 헛수고가 없었습니다. 다만 처음에 README 를 그냥 보고 "이미 flag 같은데?" 싶어서 그대로 제출했다면 `‎‎‎D‎H{...` 처럼 비표시 문자가 섞여서 **검증 시스템이 reject** 했을 가능성이 큼. 그래서:

### 5.1 회고 — "복붙하지 말고 파싱하라"

이런 워밍업 문제에서 흔히 하는 실수가 **README 의 텍스트를 그대로 복사해서 제출 입력란에 붙여넣는 것**. 비표시 문자가 함께 클립보드에 들어가 검증에 실패하면 "맞는데 왜 안 되지?" 로 시간 낭비. 처음부터 **프로그램으로 필터링한 결과** 를 복사하면 깔끔. **"보이는 것 ≠ 데이터"** 를 풀이 단계에서도 적용해야 함.

---

CTF 의 5초 컷 워밍업이지만 실무 보안에서 잊으면 큰 사고가 되는 패턴입니다. 입력 normalization, 한 번 더 점검하세요.
