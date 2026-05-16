---
layout: post
title:  "Dreamhack 워게임 [wargame.kr] already got 풀이 — 응답 헤더에 그대로 박힌 FLAG"
date:   2026-05-16 23:00:00 +0900
categories: security web wargame dreamhack writeup http-header
---

## 들어가며

Dreamhack 웹해킹 워게임 **[wargame.kr] already got** (id 1976, 풀이자 483명+, Bronze 3) 풀이.

문제 설명이 모든 걸 다 말함:

> **Can you see HTTP Response header?**

말 그대로 응답 헤더만 보면 됩니다. 풀이 전체:

```bash
$ curl -i "$URL/"
HTTP/1.1 200 OK
Server: Werkzeug/3.1.3 Python/3.13.4
Date: Sun, 17 May 2026 03:54:48 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 26
FLAG: DH{You_C@n_Mak3_Cust0m_HTTP_Head3r}
Connection: close

you've already got key! :p
```

🚩 `DH{You_C@n_Mak3_Cust0m_HTTP_Head3r}`

> Note: 정보보안 입문 학습용 PoC. 실제 운영 서비스 대상 사용 금지.

---

## 왜 이 문제가 존재하나 — HTTP 의 기본 한 번 짚기

HTTP 응답은 두 부분:

```
HTTP/1.1 200 OK              <- status line
Header-Name: value           <- headers
Another-Header: value
...                          <- 빈 줄로 헤더 끝
                             
<body content>               <- body
```

브라우저는 보통 **body 만 사용자에게 보여준다**. 헤더는 개발자 도구 > Network 탭이나 `curl -i` 같은 도구를 써야 보인다. 그래서 일반 사용자는 헤더를 신경 쓸 일이 거의 없다.

이 문제의 페이지는 body 에 `you've already got key! :p` 만 보여주는데, "이미 가지고 있다" 는 뜻 — 응답 헤더에 박혀 있으니 보러 가라는 힌트.

브라우저로 본 사람은 "어 키가 어디 있지?" 하면서 페이지 source 만 뒤지다 헤맬 가능성. `curl -i` 한 번이면 끝.

---

## 보안 교훈

- **헤더에 민감한 정보를 박지 말 것.** 브라우저는 숨겨도 클라이언트는 가져갈 수 있다. CORS, cache, 프록시, 로깅 등 다양한 경로로 노출.
- 디버그용 헤더 (`X-Debug-*`, `X-Internal-*`) 가 prod 에 새는 케이스 흔함. 응답 헤더는 다 trusted 가 아니다.
- 반대편 (공격자 입장) 에서는 **`-I`, `-i`, 브라우저 DevTools** 가 1번 reconnaissance step. body 만 보지 말고 raw response 전체를 본다.

---

## 회고

이 문제는 헛수고 0. 설명에 답이 있어서 곧장 `curl -i` 한 번.

다만 학습 포인트로:

- **문제 제목과 설명을 곧이 곧대로 읽기**. 가끔 출제자가 직접 던지는 힌트가 가장 강력하다.
- 응답 헤더 점검은 **모든 web 문제의 1단계** 로 둘 만함. 종종 `Server`, `X-Powered-By`, 커스텀 헤더에서 단서가 나옴.

🚩 `DH{You_C@n_Mak3_Cust0m_HTTP_Head3r}`
