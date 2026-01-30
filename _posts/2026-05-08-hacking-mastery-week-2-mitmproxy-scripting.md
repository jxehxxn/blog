---
layout: post
title: "Network & App Hacking Mastery: Week 2 - Scripting mitmproxy, Automating the Interception"
---

지난주, 우리는 손으로(수동으로) 패킷을 잡아서 수정해 봤습니다. 재미있었죠?
하지만 실전에서 수백, 수천 개의 패킷을 일일이 손으로 고칠 수 있을까요?
진짜 해커는 노가다를 하지 않습니다. **스크립트**를 짭니다.

오늘은 mitmproxy의 진정한 힘, **Python API**를 이용해 인터셉션을 자동화하는 방법을 배웁니다.

---

## 🤖 Why Scripting?

mitmproxy는 단순한 툴이 아닙니다. 파이썬으로 조종할 수 있는 **프레임워크**입니다.
*   특정 URL이 보이면 자동으로 데이터를 바꾼다.
*   로그인 요청이 오면 계정 정보를 파일로 저장한다.
*   서버가 보내는 응답을 조작해 에러를 유발한다.

이 모든 것을 파이썬 코드 몇 줄로 할 수 있습니다.

---

## 🔄 The Event Loop (이벤트 루프)

mitmproxy 스크립트의 핵심은 **이벤트**입니다.
데이터가 지나갈 때마다 특정 함수(이벤트)가 호출됩니다.

1.  **`request(flow)`**: 클라이언트가 서버로 요청을 보낼 때 호출됩니다.
    *   여기서 데이터를 바꾸면 서버는 바뀐 데이터를 받습니다.
2.  **`response(flow)`**: 서버가 클라이언트로 응답을 보낼 때 호출됩니다.
    *   여기서 데이터를 바꾸면 클라이언트는 바뀐 화면을 보게 됩니다.

`flow` 객체는 요청과 응답의 모든 정보를 담고 있는 택배 상자와 같습니다.

---

## 🛠️ Lab: Your First Addon (The Joker Script)

인터넷 세상을 좀 더 재밌게(혹은 혼란스럽게) 만들어 봅시다.
모든 이미지 검색 결과를 '조커' 이미지로 바꿔버리는 스크립트를 작성해 봅니다.

### 1. 스크립트 작성 (`joker.py`)

```python
from mitmproxy import http

def response(flow: http.HTTPFlow) -> None:
    # 응답 헤더의 Content-Type이 이미지인 경우만 타겟팅
    if "image" in flow.response.headers.get("content-type", ""):
        # 이미지를 내 맘대로 바꿈 (인터넷상의 웃긴 짤방 URL)
        flow.response.content = b"" # 기존 내용 삭제
        flow.response.headers["Location"] = "https://example.com/joker.jpg"
        flow.response.status_code = 302 # 리다이렉트 시킴
        print(f"🃏 Joker attacked: {flow.request.url}")
```
*   (참고: 위 코드는 리다이렉트 방식입니다. 실제 바이너리를 덮어씌우는 방식도 가능합니다.)

### 2. 실행하기
```bash
mitmweb -s joker.py
```
`-s` 옵션으로 스크립트를 로드합니다.

### 3. 결과 확인
브라우저를 열고 구글 이미지를 검색해 보세요. 세상이 조커로 가득 찼나요?

---

## 🏢 Big Tech Practice

빅테크 기업에서는 이 기능을 어떻게 활용할까요?
단순 장난이 아니라, **QA(품질 보증)와 보안 테스트**에 사용합니다.

*   **Error Injection:** 결제 승인 요청이 갈 때, 강제로 `500 Internal Server Error`를 리턴하게 스크립트를 짭니다. 앱이 죽지 않고 "일시적인 오류입니다"라고 우아하게 처리하는지 테스트합니다.
*   **Security Regression:** 특정 파라미터에 자동으로 SQL Injection 구문을 넣어서 보내는 퍼징(Fuzzing) 도구로 활용합니다.

---

## 📝 2주차 과제: JSON 수집기 만들기

여러분이 방문하는 모든 사이트 중, 응답이 **JSON** 형식인 경우를 찾아서
별도의 파일(`loot.json`)에 저장하는 스크립트를 작성하세요.

**힌트:**
1.  `response(flow)` 함수를 사용하세요.
2.  `flow.response.headers["Content-Type"]`이 `application/json`인지 확인하세요.
3.  파이썬의 `open("loot.json", "a")`를 사용해 파일에 `flow.response.text`를 기록하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** mitmproxy 스크립트에서, 클라이언트가 서버로 데이터를 보낼 때 호출되는 이벤트 함수의 이름은?
2.  **Q2.** 스크립트를 mitmproxy와 함께 실행할 때 사용하는 커맨드라인 옵션은 무엇인가요? (알파벳 한 글자)
3.  **Q3.** `flow` 객체 안에서 현재 요청의 URL을 가져오려면 어떤 속성을 참조해야 할까요? (`flow.request.???`)

---

다음 주는 **Advanced Traffic Manipulation**입니다.
단순히 내용을 바꾸는 것을 넘어, 요청을 복제해서 100번 다시 보내는(Replay) 공격과
다른 사람의 쿠키를 훔쳐서 신분을 위장하는 고급 기법을 다룹니다.

**Automate everything.**
