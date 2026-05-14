---
layout: post
title:  "Dreamhack 워게임 Click me! 풀이 — 도망다니는 버튼을 굳이 마우스로 잡지 말자"
date:   2026-05-15 04:00:00 +0900
categories: security web wargame dreamhack writeup javascript
---

## 들어가며

Dreamhack 워게임 **Click me!** (난이도 Bronze 4) 풀이입니다. 문제 설명은 단 한 줄.

> Catch me if you can!

ZIP 안의 README도 한 줄짜리 농담입니다.

> Just click the button! Then you win the flag.
> 버튼을 클릭하시면 됩니다! 플래그를 드립니다.

이 문제의 본질은 **"클라이언트에서 일어나는 일은 다 가짜다"** 라는 웹해킹 입문 1교시 명제를 체험시키는 것입니다.

스포일러 결론: **버튼을 클릭할 필요 자체가 없습니다.** `curl` 한 줄로 끝납니다.

```bash
curl "$SERVER/fnxmtmznpdjakstp"
# → Good! Flag is DH{How_did_you_click_this_button?}
```

## 1. 페이지 첫인상

VM 부팅 후 `/` 에 접속하면 가운데에 빨간 **Click me!** 버튼이 떠 있고, 마우스 커서를 가까이 가져가는 순간 버튼이 화면 다른 곳으로 도망갑니다. 손으로 잡으려고 하면 영원히 못 잡는 짓궂은 UI입니다.

## 2. 소스 한 줄씩 읽기

```html
<button id="escapeButton" tabindex="-1" onclick="alert('You Clicked Me!')">
  Click me!
</button>

<script>
  const escapeButton = document.getElementById("escapeButton");
  const ESCAPE_DISTANCE = 200;

  moveButtonRandomly();

  document.addEventListener("mousemove", (event) => {
    // 마우스가 ESCAPE_DISTANCE 안으로 들어오면 버튼 위치 랜덤 이동
    if (distance < ESCAPE_DISTANCE) {
      moveButtonRandomly();
    }
  });

  function moveButtonRandomly() { /* style.left, style.top 랜덤 */ }

  escapeButton.addEventListener("click", () => {
    window.location.href = "/fnxmtmznpdjakstp";    // ★ 클릭 시 이동할 진짜 엔드포인트
  });
</script>
```

세 가지 사실이 보입니다.

1. 버튼이 도망다니는 것은 **시각적 장난** 일 뿐 — 서버 측 검증은 전혀 없음.
2. 클릭 핸들러는 **고정된 URL `/fnxmtmznpdjakstp`** 로 페이지를 이동시키는 게 전부.
3. 그 URL이 응답으로 주는 게 flag 일 것이다.

브라우저는 보안 결정의 주체가 아닙니다. 그저 GET 요청을 보내는 도구일 뿐. 그 URL을 안다면 누가/무엇이 요청을 보내든 상관이 없습니다.

## 3. 풀이

```bash
$ SERVER=http://host3.dreamhack.games:22733
$ curl "$SERVER/fnxmtmznpdjakstp"
Good! Flag is DH{How_did_you_click_this_button?}
```

flag: `DH{How_did_you_click_this_button?}` ✓

## 4. 입문자가 가져갈 교훈

이 문제는 5초 컷이지만, 그 5초가 가르치는 것은 의외로 크고 자주 잊혀집니다.

### 4-1. 클라이언트 검증은 검증이 아니다

`escapeButton.addEventListener("click", ...)` 안에 무슨 로직을 넣어도, **그건 그저 그 브라우저의 그 페이지 안에서만 작동하는 코드**입니다. 서버가 `/fnxmtmznpdjakstp` 에 어떤 검증도 안 걸어두면, 누구든 그 URL을 직접 호출할 수 있습니다.

실무에서 이 클래스의 사고는 매년 반복됩니다.

- 쇼핑몰의 "가격이 0원이 안 되게" 검증을 JavaScript 로만 → 클라이언트가 가격 0원으로 직접 POST → 결제 통과
- 게임 클라이언트의 "쿨다운 60초" 검증을 JS 로만 → 봇이 쿨다운 무시하고 RPC 직접 호출
- 어드민 페이지의 "권한 확인 후 버튼 노출" 만 처리 (서버는 미검증) → URL 직접 입력으로 권한 우회

### 4-2. UI 트릭과 보안은 무관하다

도망다니는 버튼, 무한 captcha, 마우스 시간 측정 — 모두 **봇 차단/사용성 장치**일 뿐, **권한이나 정합성을 강제하지 않는다**. 보안 모델을 짤 때 "사용자가 화면에 보이는 대로만 행동할 것이다" 라는 가정은 항상 틀린다.

### 4-3. F12 가 답이다

이런 문제를 만나면 가장 먼저 할 일은 **DevTools → Sources(또는 페이지 소스 보기)**. 클라이언트 JS 안에 정답 URL/엔드포인트가 박혀있는 경우가 압도적으로 많다. 마우스로 해결하려 하지 말고 코드를 읽자.

```bash
# 또는 명령 한 줄로
curl -s "$SERVER/" | grep -oE 'window\.location\.href\s*=\s*"[^"]+"'
# → window.location.href = "/fnxmtmznpdjakstp"
```

### 4-4. 방어 코드 한 줄

이 문제를 출제자가 진짜로 막고 싶었다면 서버에서 다음과 같이 처리하면 됩니다.

```python
@app.route('/fnxmtmznpdjakstp', methods=['POST'])
def flag():
    if not is_valid_session(request):
        abort(403)
    if not request.headers.get('X-CSRF') == session['csrf']:
        abort(403)
    # ... 추가 검증
    return f"Flag is {FLAG}"
```

POST 강제 + 세션/CSRF 검증 + (필요시) rate-limit. UI는 보조 수단으로만 두면 됩니다.

## 5. 정리

- 클라이언트에서 서버로 가는 통로는 결국 HTTP 요청. **그 통로를 직접 두드릴 수 있다면 클라이언트 UI는 무관**하다.
- 문제든 실무든, **JS 소스의 모든 URL/엔드포인트/플래그/토큰** 은 일단 grep 으로 추출해 보자. 정답이 거기 있는 경우가 많다.
- "Catch me if you can" 같은 문제 설명은 보통 "마우스로 안 잡혀도 다른 방법이 있다"는 자조적인 힌트다.

---

## 부록 — 시도하다 막혔던 지점과 회고

### 실패 1. 브라우저로 진짜 클릭해보려 함

처음에 무심코 페이지를 열고 마우스로 버튼을 추격해 봤습니다. 이걸 진지하게 시도하는 그 자체가 **이 문제의 함정**이라는 점을 늦지 않게 깨달았습니다. 코드를 보는 게 항상 빠르다.

**회고**: 워게임/실무 안 가리고, 처음 30초는 **UI 만지지 말고 소스부터 본다**. 특히 "버튼/UI 가 핵심으로 보이는" 문제일수록 더 그래야 한다 — 그게 함정이다.

### 실패 2. JS 시뮬레이션을 잠시 고민

브라우저 콘솔에서 `escapeButton.click()` 을 호출해 자바스크립트 이벤트로 클릭을 강제할까도 생각했습니다. 작동했겠지만 의미가 없습니다. `addEventListener("click", ...)` 핸들러가 결국 `/fnxmtmznpdjakstp` 로 navigation 을 하는 거니까, **그 URL을 알아내자마자 클릭이라는 단계 자체가 의미가 없어집니다**.

**회고**: "어떻게 우회할까?" 보다 "**무엇을 정말 우회해야 하는가?**" 를 먼저 묻자. 우회 대상이 클라이언트 단의 시각적 장치라면 그건 우회할 가치도 없는 장치다.

### 실패 3. 다운로드 ZIP 이 너무 작아서 잠시 의심

ZIP 크기가 316바이트라 처음에는 "받다가 끊긴 거 아니야?" 라고 잠깐 의심했습니다. 풀어보니 README 한 줄짜리 파일. 출제자가 일부러 "ZIP 에는 단서가 거의 없다 — 라이브 서버를 봐라" 라고 알려주는 신호였습니다.

**회고**: ZIP 크기가 작으면 의심하기 전에 일단 풀어 보자. 그리고 ZIP 에 단서가 없으면 라이브 서버에 단서가 있다는 뜻이다.
