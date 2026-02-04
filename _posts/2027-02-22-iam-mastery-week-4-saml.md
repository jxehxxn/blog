---
layout: post
title: "IAM Mastery with Authentik: Week 4 - The Enterprise Handshake: SAML 2.0 Deep Dive"
---

"우리 회사는 보안 때문에 내부망에서만 동작하는 레거시 ERP를 씁니다. OIDC 지원 안 한대요."
"공공기관이랑 연동해야 하는데 SAML 명세서를 던져주네요."

스타트업이나 모던 웹 환경에서는 OIDC가 왕이지만, **엔터프라이즈와 공공 부문에서는 여전히 SAML 2.0이 지배자**입니다. XML 기반이라 무겁고 어렵지만, 기능은 강력합니다. Authentik은 매우 훌륭한 SAML IdP입니다.

---

## 1. What is SAML? (Security Assertion Markup Language)

2005년에 만들어진 XML 기반의 표준입니다.
OIDC가 "JSON으로 토큰 주고받기"라면, SAML은 **"XML로 작성된 신임장(Assertion)을 브라우저를 통해 전달하기"**입니다.

### Key Terminology
용어부터 다릅니다. OIDC와 매핑해봅시다.
*   **IdP (Identity Provider):** Authentik. (OIDC의 OP)
*   **SP (Service Provider):** 애플리케이션 (Jenkins, VPN, AWS Console). (OIDC의 RP)
*   **Assertion:** 인증 정보가 담긴 XML 덩어리. (OIDC의 ID Token)
*   **ACS URL (Assertion Consumer Service):** SP가 Assertion을 받는 주소. (OIDC의 Redirect URI)
*   **Entity ID:** IdP와 SP를 식별하는 고유 ID. (OIDC의 Client ID/Issuer URL)

---

## 2. The SAML Flow (SP-Initiated)

가장 흔한 흐름입니다.

1.  **User -> SP:** "로그인 할래."
2.  **SP -> User:** "이 **SAML Request(XML)**를 가지고 IdP로 가." (Redirect)
3.  **User -> Authentik (IdP):** (가져온 XML을 보여주며) 로그인 시도.
4.  **Authentik:** 인증 성공. "자, 여기 네 정보가 담긴 **SAML Response(XML + 서명)**야."
5.  **User -> SP:** (Authentik이 시킨 대로) SP의 ACS URL로 **POST 요청**을 보냄. (XML을 Body에 담아서)
6.  **SP:** XML 서명을 검증하고 로그인 시킴.

**특징:** 모든 데이터가 사용자의 브라우저를 거쳐갑니다(Front-channel). 서버 간 직접 통신(Back-channel)이 거의 없습니다.

---

## 3. Metadata Exchange: The Trust Setup

OIDC는 Client ID/Secret만 알면 되지만, SAML은 **"신뢰 관계(Trust Relationship)"** 구축이 까다롭습니다.
서로의 **Metadata XML 파일**을 교환해야 합니다.

*   **IdP Metadata:** "내 로그인 URL은 이거고, 내 공개키(인증서)는 이거야."
*   **SP Metadata:** "내 ACS URL은 이거고, 내 Entity ID는 이거야."

Authentik에서는 "Provider" 설정에서 메타데이터 URL을 제공합니다. 이걸 복사해서 SP(예: AWS IAM)에 붙여넣어야 합니다.

---

## 4. Attributes Mapping

SAML의 악몽은 **속성 이름 맞추기**입니다.

*   Authentik: `email`
*   어떤 SP: `urn:oid:0.9.2342.19200300.100.1.3` (???)
*   다른 SP: `User.Email`

표준이 있긴 하지만 잘 안 지킵니다.
Authentik의 **"Property Mapping"** 기능이 여기서 빛을 발합니다. "내 내부 `user.email`을 SP에게 줄 때는 `User.Email`이라는 이름으로 포장해서 줘라"라고 설정할 수 있습니다.

---

## 🛠️ Lab: SAML Tracer Analysis

SAML은 디버깅이 어렵습니다. 브라우저에서 암호화된(Base64) XML이 오가기 때문입니다.
Chrome 확장 프로그램인 **"SAML Tracer"**나 **"SAML DevTools"**가 필수입니다.

### 시나리오
1.  SAML Tracer를 켭니다.
2.  (가능하다면) Authentik 자체를 SP로 설정하거나, 무료 SAML 테스트 사이트(SAML.to 등)를 이용해 로그인을 시도합니다.
3.  노란색으로 표시된 SAML 패킷을 찾습니다.
4.  **Decode** 탭을 눌러 XML을 봅니다.
    *   `<saml:NameID>`: 내 아이디가 맞는지?
    *   `<saml:AttributeStatement>`: 내가 설정한 속성(Email, Name)이 들어있는지?
    *   `Signature`: 서명이 유효한지?

---

## 📝 4주차 과제: AWS Console SSO with SAML

(실제 AWS 계정이 없다면 과정만 서술하세요)

**목표:** Authentik을 이용해 AWS Management Console에 로그인하는 설정을 구상하세요.

1.  **AWS IAM:** Identity Provider 메뉴에서 Authentik의 메타데이터 XML을 업로드합니다.
2.  **Authentik:** AWS를 위한 Provider/Application을 생성합니다.
3.  **Attribute Mapping:** AWS는 특수한 속성 2가지를 요구합니다.
    *   `https://aws.amazon.com/SAML/Attributes/Role`: 로그인할 IAM Role ARN.
    *   `https://aws.amazon.com/SAML/Attributes/RoleSessionName`: 세션 이름(보통 이메일).
4.  이 두 속성을 Authentik에서 어떻게 매핑할지 계획을 세우세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** SAML 통신 과정에서 사용자의 신원 정보가 담긴 XML 메시지를 무엇이라 부르나요? (Assertion)
2.  **Q2.** SP가 IdP에게 인증 요청을 보낼 때, 또는 IdP가 SP에게 응답을 보낼 때 사용하는 HTTP 메소드는 주로 무엇인가요? (GET-Redirect 또는 POST-Binding)
3.  **Q3.** OIDC와 달리 SAML은 모바일 앱(Native App)에서 구현하기가 까다롭습니다. 그 이유는? (XML 파싱의 무거움 + 웹뷰 의존성 + 복잡한 암호화)

---

이것으로 지루하지만 필수적인 **"프로토콜 3대장(LDAP, OIDC, SAML)"** 학습이 끝났습니다.
이제 기초 체력은 다졌습니다. 다음 주, 드디어 주인공 **Authentik**을 설치하고 이 모든 것을 통합해봅니다.

**XML is not dead (yet).**
