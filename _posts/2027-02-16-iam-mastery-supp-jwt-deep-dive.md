---
layout: post
title: "IAM Mastery with Authentik: Supplement - OAuth Grant Types & JWT Anatomy"
---

Week 3에서 OIDC의 표준 흐름인 `Authorization Code Flow`를 배웠습니다. 하지만 세상엔 다양한 환경이 존재합니다. 브라우저가 없는 서버, 화면이 없는 IoT 기기, 신뢰할 수 없는 모바일 앱...

상황에 맞춰 OAuth는 다양한 **Grant Type(토큰 발급 방식)**을 제공합니다. Authentik 설정 화면에서 "Grant Type" 체크박스를 볼 때 당황하지 않도록, 핵심 타입을 정리해드립니다.

---

## 1. OAuth 2.0 Grant Types: 상황별 전략

### 1) Authorization Code Grant (PKCE)
*   **용도:** 대부분의 웹 앱, 모바일 앱, SPA(Single Page App).
*   **특징:** 가장 안전함. `client_secret`을 쓸 수 없는 모바일/SPA 환경을 위해 **PKCE(Proof Key for Code Exchange)**라는 추가 보안책을 사용합니다.
*   **Authentik 설정:** 기본값. 무조건 이걸 쓰세요.

### 2) Client Credentials Grant
*   **용도:** 사용자 개입 없이 **서버끼리** 통신할 때 (M2M - Machine to Machine).
*   **예시:** "정산 서버"가 "은행 API"를 호출할 때. 로그인 창 안 뜸. 그냥 ID/Secret 주고 토큰 받음.
*   **주의:** 이 방식으로는 사용자 정보(ID Token)를 못 받습니다. 앱(Client) 자체가 주체가 되기 때문입니다.

### 3) Implicit Grant (Deprecated)
*   **용도:** 옛날 SPA.
*   **특징:** Code 교환 없이 바로 URL 파라미터로 Access Token을 줍니다.
*   **보안:** **쓰지 마세요.** 브라우저 히스토리에 토큰이 남습니다. 요즘은 PKCE가 표준입니다.

### 4) Resource Owner Password Credentials Grant (Legacy)
*   **용도:** 진짜 어쩔 수 없는 레거시 앱.
*   **특징:** 앱이 사용자에게 ID/PW를 직접 입력받아서 Authentik에 던집니다.
*   **보안:** **최악.** 앱이 사용자 비밀번호를 알게 됩니다. OAuth의 철학(비번 공유 X)을 위배합니다. Authentik은 지원하긴 하지만, 정말 피치 못할 때만 쓰세요.

---

## 2. JWT (JSON Web Token) Anatomy

OIDC의 ID Token이나, 일부 API의 Access Token은 JWT 포맷입니다.
`aaaaa.bbbbb.ccccc` 처럼 점(.) 두 개로 구분된 세 덩어리 문자열입니다.

### Header (빨간색)
*   `typ`: "JWT"
*   `alg`: 서명 알고리즘 (예: `RS256`, `HS256`)
    *   **RS256:** 비대칭 키. Authentik이 개인키로 잠그고, 앱은 공개키로 엽니다. (추천)
    *   **HS256:** 대칭 키. Authentik과 앱이 같은 비밀번호를 공유합니다.

### Payload (보라색)
실제 데이터(Claims)가 들어갑니다. Base64로 인코딩되어 있을 뿐, **암호화된 게 아닙니다.**
누구나 디코딩해서 볼 수 있습니다. **절대 비밀번호 같은 민감 정보를 넣지 마세요.**

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "groups": ["admin", "dev"]
}
```

### Signature (파란색)
`Hash(Header + Payload, Secret Key)` 값입니다.
누군가 Payload의 `groups`를 `admin`으로 조작하면? Signature가 틀려서 검증에 실패합니다. 이것이 JWT의 무결성을 보장합니다.

---

## 3. JWT vs Opaque Token

*   **JWT (By-value Token):** 토큰 자체가 정보를 담고 있음. DB 조회 없이 검증 가능(Stateless). 토큰이 큼.
*   **Opaque Token (By-reference Token):** 그냥 무작위 문자열(예: `7f3a1...`). 정보를 알으려면 발급처(Authentik)에 물어봐야 함(Introspection). 보안성이 높고 즉시 폐기(Revoke) 가능.

Authentik은 주로 JWT를 쓰지만, 보안 요구사항에 따라 Access Token을 Opaque하게 줄 수도 있습니다.

---

## 4. Troubleshooting JWT

Authentik 연동 시 "Invalid Token" 에러가 난다면?

1.  **시간 동기화:** Authentik 서버와 앱 서버의 시간이 다르면 `iat`(발행 시간)나 `exp`(만료 시간) 검증에서 실패합니다. NTP를 확인하세요.
2.  **Audience (`aud`) 불일치:** 토큰은 "이 토큰은 App-A 전용이야"라고 적혀있는데 App-B가 쓰려고 하면 거부됩니다.
3.  **Signing Key:** Authentik은 주기적으로 서명 키를 바꿀 수 있습니다(Rotation). 앱이 옛날 공개키를 가지고 있으면 검증 못 합니다. 앱은 JWKS URL(`/.well-known/jwks.json`)을 통해 최신 키를 받아와야 합니다.

토큰은 거짓말을 하지 않습니다. 까보면(Decode) 답이 나옵니다.
