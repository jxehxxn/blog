---
layout: post
title:  "Dreamhack 워게임 PEPERO 풀이 — 음수 수량 입력으로 부자가 되는 쇼핑몰 비즈니스 로직 버그"
date:   2026-05-20 12:30:00 +0900
categories: security web wargame dreamhack writeup business-logic integer-validation
---

## 들어가며

Dreamhack 웹해킹 워게임 **PEPERO** (난이도 별 1, 풀이자 369명) 풀이입니다. 빼빼로 데이 기념 미니 쇼핑몰 챌린지. 소스 없이 **검은 상자 (black-box) UI 만 주어진** 가벼운 비즈니스 로직 문제예요. 핵심 한 줄: **`amount` 입력값이 음수일 때를 서버가 검증하지 않아서, 빼빼로를 "음수 개" 사면 돈이 늘어난다**.

문제 설명:

> 빼빼로 데이는 드림핵과 함께!
> 빼빼로 가게에 어서 오세요!!

스포일러 결론:

```bash
SERVER="http://host3.dreamhack.games:PORT"
# 시작: 소지금 1000원, 빼빼로 0개, 플래그 가격 2000원
curl -c jar -b jar "$SERVER/"

# 음수 수량으로 "구매" → 환불 받음 (소지금 8000원 증가)
curl -c jar -b jar -X POST "$SERVER/buy_pepero" -d "amount=-100"

# 이제 9000원, 플래그 구매 가능
curl -c jar -b jar -X POST "$SERVER/buy_flag"
curl -c jar -b jar "$SERVER/"
# → DH{Rootsquare_loves_pepero}
```

5초 컷. **사용자 입력 검증의 가장 클래식한 빈틈** 한 가지.

## 1. 자료 파악 — README 한 장이 전부

ZIP 안에는 `README.md` 한 줄짜리만:

> Welcome to Pepero shop! We want to buy a flag.

소스가 없으니 black-box. 일단 사이트를 직접 둘러봅니다.

서버를 켜고 `/` 에 접속하면 보이는 UI:

```
💰 현재 소지금: 1000원
🍫 빼빼로 보유 개수: 0개

[ 구매할 개수 입력 (100원/개) ]  [🍫 빼빼로 사기]
                                  [💡 힌트 사기 (빼빼로 10개)]
                                  [🚩 플래그 사기 (2000원)]
```

해석:

- 시작 자본: **1000원**.
- 빼빼로 한 개 가격: **100원**.
- 플래그 가격: **2000원**.
- 힌트는 빼빼로 10개 (= 1000원어치) 가 필요.

→ 1000원으로 시작해서 어떻게든 **2000원을 만들어야** 플래그를 살 수 있다. 정상 경로로는 절대 도달 못 함 (가격이 늘기만 할 뿐 돈이 늘 길은 안 보임). 그러면 **비정상 입력으로 돈을 늘리는 방법** 을 찾아야 함.

## 2. 1차 가설 — 입력 검증의 표준 함정 후보들

쇼핑몰 비즈니스 로직 문제에서 항상 점검하는 input validation 후보들을 머릿속에 꺼냅니다:

| 입력 시도 | 노리는 버그 |
|---|---|
| `amount = -100` | 음수 → 환불처럼 동작해서 돈 늘어남 |
| `amount = 0` | 무료 결제 가능? |
| `amount = 0.5` | 부동소수 → 0원짜리 구매? |
| `amount = 1e10` | 큰 숫자 overflow / 음수 wrap |
| `amount = abc` | 타입 confusion → NaN |
| `amount = 1; amount = -1` | parameter pollution |

`<input type="number" required min="0">` 가 **HTML 단에서 음수를 막고는** 있지만, 그건 클라이언트단. 서버가 다시 검증하지 않으면 곧장 통과. 가장 먼저 음수부터 시도.

## 3. 익스플로잇

### 3.1 음수 수량으로 환불 효과

```bash
curl -c jar -b jar -X POST "$SERVER/buy_pepero" -d "amount=-100"
```

이후 `/` 를 확인하면:

```
💰 현재 소지금: 9000원
🍫 빼빼로 보유 개수: -100개
```

소지금이 1000 → **9000** 으로 점프. (1000 + 8000 = 9000. 정확한 산식은 서버가 어떻게 구현했든 음수에 의해 잘못 더해진 것.)

> 추측 산식: `money += -amount * 100` (구매할 때 줄어드는 부분에 부호 반대 처리 한 줄 빠진 듯) 또는 `money = money - amount * 100` 가 `amount=-100` 일 때 `1000 - (-100)*100 = 1000 + 10000 = 11000` 인데 9000 이 나오는 걸 보면 일부 클램프나 다른 계산식. 어쨌든 **돈이 늘었으니 우리 목적엔 충분**.

### 3.2 플래그 구매

```bash
curl -c jar -b jar -X POST "$SERVER/buy_flag"
curl -c jar -b jar "$SERVER/"
# → DH{Rootsquare_loves_pepero}
```

`/buy_flag` 가 2000원 이상이면 통과시키도록 만들어져 있고, 우리는 9000원이라 무난히 통과. 홈 화면이 자동으로 플래그를 노출.

## 4. 왜 이런 버그가 생기는가

비즈니스 로직 (특히 결제/쿠폰/포인트/적립금) 에서 **음수 / 0 / 매우 큰 값** 검증을 누락하는 건 흔한 패턴입니다. 흔한 원인 3가지:

1. **클라이언트 검증에만 의존**: `<input type="number" min="0">` 으로 막아놨으니 사용자가 양수만 보낼 거라고 가정. 하지만 HTML attribute 는 **순수 UX**, 보안 아니다.
2. **양수 흐름만 테스트**: QA 도 사람이라 "예상 사용 흐름" 위주로 테스트. 음수, 빈 값, 매우 큰 값, 0 같은 경계는 자주 빠짐.
3. **'환불' 과 '판매' 가 같은 함수로 구현**: 한 함수에서 부호로만 구분하는 경우, 결제 권한 검사 (관리자만 환불 가능) 가 사라지면 곧장 무료 환불 가능.

실제 사고 사례:

- 2015년경 한 국내 게임 인앱 상점에서 음수 수량으로 결제하면 재화가 늘어나는 버그.
- 2019년 한 페이먼트 API 가 `amount: -100` 으로 charge 호출 시 환불 처리되어 임의 카드에서 출금되는 사고.
- "쿠폰 코드 적용 시 가격이 음수가 되면 그만큼 돈을 받는다" 류의 흔한 e-commerce 함정.

## 5. 안전하게 고치기

서버 측에서 **모든 수치 입력의 도메인을 명시적으로 좁히는** 게 정공법.

```python
@app.route("/buy_pepero", methods=["POST"])
def buy_pepero():
    try:
        amount = int(request.form["amount"])
    except (TypeError, ValueError):
        abort(400, "invalid amount")
    if amount <= 0:
        abort(400, "amount must be positive")
    if amount > 100:               # 일회 구매 한도
        abort(400, "amount too large")

    cost = amount * 100
    if session["money"] < cost:
        abort(400, "not enough money")

    session["money"] -= cost
    session["pepero"] += amount
    return redirect("/")
```

핵심 4가지:

1. **타입 캐스팅 + 예외 처리**. 비정형 입력은 즉시 400.
2. **음수/0 명시 차단**.
3. **상한선** (overflow 방지 + 비즈니스 한도).
4. **잔액 검증 분기**.

이 4줄이 비즈니스 로직 보안의 99% 를 커버합니다.

## 6. 한 번 더 정리

### 6.1 취약점 카탈로그

| 단계 | 카테고리 | 원인 |
|---|---|---|
| `amount=-100` 허용 | Business Logic / Improper Input Validation (CWE-20) | 서버가 음수 검증 안 함 |
| 클라이언트 `min="0"` 만 존재 | Reliance on Client-Side Validation | UX 와 보안 혼동 |
| 환불 권한 검증 없음 | Missing Authorization | "환불" 흐름이 일반 결제와 분리되지 않음 |

### 6.2 입문자가 챙겨가면 좋은 시각

- 결제/포인트/적립 시스템을 보면 **자동으로 -1, 0, 매우 큰 수, 부동소수, 문자열** 을 모두 던져봐라. 30초.
- **HTML attribute 는 무시해도 된다는 가정**: `<input type="email">`, `<input min="0">`, `<select>`, `disabled` 모두 클라이언트가 우회 가능. 보안은 항상 서버에서.
- 비즈니스 로직 문제에서 소스가 없을 때, **각 액션을 호출했을 때 변하는 state 의 산식을 역추정** 하는 습관. 1000 → 9000 으로 점프하면 어떤 산식이 음수와 만나 그렇게 됐을지 머릿속에서 모델링 → 다른 endpoint 에서도 활용 가능.

## 7. 시도했지만 실패한 것들 (회고)

이번엔 첫 시도가 정답이라 별도 헛수고가 없었습니다. 다만 처음 black-box 를 본 순간 "소스 없이 풀려면 시간 걸리겠다" 는 약간의 멈칫이 있었음.

**회고**: 비즈니스 로직 챌린지는 소스 있는 코드 리뷰보다 **빠르게 입력을 변형해서 던지는** 방식이 더 효율적인 경우가 많음. 모든 입력 필드에 음수/0/오버플로/문자열/배열을 30초씩 시도해 보는 게 first pass.

특히 black-box 일수록 **소스가 없어서 어렵다** 가 아니라 **소스가 없어서 가설이 단순해도 된다** 로 바꿔 생각하자. UI 가 보여주는 액션의 수가 곧 공격 표면의 수.

---

`<input type="number" min="0" required>` 는 사용자에게 친절한 UX 일 뿐, 공격자의 음수 입력을 막아주지 않습니다. 모든 검증은 서버에서.
