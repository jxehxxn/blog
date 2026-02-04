---
layout: post
title: "IAM Mastery with Authentik: Week 3 - The Handshake: OAuth 2.0 & OIDC Deep Dive"
---

Week 2에서 우리는 "사용자 정보를 어디에 저장하는가(LDAP)"를 배웠습니다.
이제 그 정보를 가지고 "어떻게 다른 사이트에 로그인하는가"를 배울 차례입니다.

"Google로 로그인", "Facebook으로 로그인"... 너무나 익숙하죠?
이 뒤에는 **OAuth 2.0**과 **OpenID Connect (OIDC)**라는 거대한 프로토콜이 숨어 있습니다. Authentik은 훌륭한 OIDC Provider입니다. 원리를 알아야 설정할 수 있습니다.

---

## 1. OAuth 2.0: The Valet Key (발렛 파킹 키)

OAuth 2.0은 원래 **"권한 위임(Authorization)"** 프로토콜입니다.
*   **시나리오:** 내가 '사진 인화 앱'을 씁니다. 이 앱이 내 'Google Photos'에 있는 사진을 가져와야 합니다.
*   **문제:** 앱에 내 Google 비번을 알려주기 싫습니다.
*   **해결:** Google에게 "이 앱이 내 사진만 볼 수 있게(Scope) 임시 키(Access Token) 하나만 줘"라고 부탁합니다.

이것이 OAuth입니다. **비밀번호 공유 없이 권한만 빌려주는 것**입니다.

### Roles
*   **Resource Owner:** 나 (사용자).
*   **Client:** 사진 인화 앱 (Authentik 입장에서는 연동할 애플리케이션).
*   **Authorization Server:** Google (Authentik).
*   **Resource Server:** Google Photos API.

---

## 2. OpenID Connect (OIDC): Authentication Layer

사람들은 OAuth를 보고 생각했습니다. "이걸로 로그인도 시키면 안 되나?"
하지만 OAuth Access Token은 "열쇠"지 "신분증"이 아닙니다. 열쇠만 보고 이 사람이 누군지 알 수 없죠.

그래서 OAuth 2.0 위에 얇은 막을 하나 더 얹었습니다. 그게 **OIDC**입니다.
*   **OAuth 2.0:** Access Token을 줌. (API 호출용)
*   **OIDC:** **ID Token**을 추가로 줌. (신분 확인용)

ID Token은 **JWT(JSON Web Token)** 형식으로 되어 있으며, 안에 "이 사람은 홍길동이고, 이메일은 a@b.com이다"라는 정보가 서명되어 들어있습니다.

---

## 3. The Authorization Code Flow (가장 중요한 흐름)

보안상 가장 권장되는 표준 흐름입니다. Authentik에서 앱을 연동할 때 이 흐름을 씁니다.

1.  **User -> App:** "로그인 할래."
2.  **App -> Authentik:** 사용자를 Authentik 로그인 페이지로 보냄. (`response_type=code`)
3.  **User -> Authentik:** ID/PW 입력하고 로그인.
4.  **Authentik -> App:** "자, 여기 **Authorization Code** (임시 교환권) 줄게." (리다이렉트)
5.  **App -> Authentik (Back-channel):** "아까 받은 코드(교환권)랑 내 비밀키(Client Secret) 줄게. 진짜 토큰 줘."
6.  **Authentik -> App:** "확인됐어. 여기 **Access Token**이랑 **ID Token** 받아."
7.  **App:** ID Token을 까서 사용자 정보를 확인하고 로그인 시킴.

왜 이렇게 복잡하게 할까요? **사용자의 브라우저(Front-channel)에 Access Token을 노출시키지 않기 위해서**입니다. (보안!)

---

## 4. Scopes & Claims

*   **Scope:** "어디까지 보여줄래?"
    *   `openid`: 필수. 나 OIDC 할거야.
    *   `profile`: 이름, 별명 등 기본 프로필.
    *   `email`: 이메일 주소.
    *   `offline_access`: Refresh Token 주세요. (로그인 유지)
*   **Claim:** "토큰 안에 들어있는 정보 조각"
    *   `sub` (Subject): 사용자 고유 ID.
    *   `iss` (Issuer): 누가 발급했나 (Authentik URL).
    *   `exp` (Expiration): 만료 시간.

Authentik의 강력함은, 이 **Scope와 Claim을 마음대로 커스터마이징** 할 수 있다는 데 있습니다. "부서(department)" 정보를 토큰에 넣고 싶다면? Authentik에서 매핑만 하면 됩니다.

---

## 🛠️ Lab: OIDC Debugger Trace

직접 눈으로 토큰을 까봅시다.

1.  [OIDC Debugger](https://oidcdebugger.com/) 사이트에 접속합니다.
2.  (만약 Authentik이 설치되어 있다면) Authentik을 설정하고 연결해봅니다.
3.  없다면 Google의 데모 설정을 이용해 Flow를 타봅니다.
4.  결과로 받은 **ID Token**을 복사해서 [jwt.io](https://jwt.io)에 붙여넣어 보세요.
5.  Header, Payload, Signature가 어떻게 생겼는지 확인하세요.

---

## 📝 3주차 과제: OIDC Manual Flow

**목표:** 라이브러리 없이 `curl`이나 `Postman`만으로 OIDC Flow를 단계별로 수행하세요.

1.  **Authorize Request:** 브라우저에 URL을 쳐서 Authorization Code를 받아오세요.
2.  **Token Request:** `curl -X POST`로 코드를 Access Token/ID Token으로 교환하세요.
3.  **Userinfo Request:** 받은 Access Token을 헤더에 넣고 Userinfo 엔드포인트를 찔러서 JSON 정보를 받아오세요.

이 과정을 거치면 OIDC 라이브러리가 마법처럼 해주던 일이 사실은 단순한 HTTP 요청이었음을 깨닫게 됩니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** OAuth 2.0은 인증(Authentication) 프로토콜인가요, 인가(Authorization) 프로토콜인가요? (인가 - Authorization)
2.  **Q2.** OIDC에서 사용자 정보를 담고 있는 JWT 형태의 토큰 이름은? (ID Token)
3.  **Q3.** 브라우저를 통하지 않고, 서버끼리 직접 통신하여 토큰을 교환하는 채널을 무엇이라 하나요? (Back-channel)

---

다음 주, 우리는 조금 더 올드하지만, 엔터프라이즈의 거물인 **SAML 2.0**을 만납니다. XML의 늪으로 갈 준비 되셨나요?

**Tokenize everything.**
