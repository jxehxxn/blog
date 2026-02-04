---
layout: post
title: "IAM Mastery with Authentik: Week 6 - The Engine: Flows, Stages, and Policies"
---

Authentik이 다른 IAM(Keycloak 등)과 가장 차별화되는 점은 바로 **Flow** 기반의 유연한 아키텍처입니다.
"로그인할 때 MFA를 먼저 보여줄까, 나중에 보여줄까?"
"특정 그룹만 동의 약관을 띄울까?"

Authentik에서는 이 모든 것을 코딩 없이, 블록 조립하듯이 만들 수 있습니다.

---

## 1. Core Concepts: Flow, Stage, Prompt

### Flow (흐름)
사용자가 경험하는 전체 여정입니다.
*   **Authentication Flow:** 로그인 과정.
*   **Enrollment Flow:** 회원가입 과정.
*   **Recovery Flow:** 비밀번호 찾기 과정.

Flow는 URL(`.../if/flow/my-login-flow/`)로 접근 가능하며, 여러 개의 Stage로 구성됩니다.

### Stage (단계)
Flow 안에서 실행되는 하나의 작업 단위입니다.
*   **Identification Stage:** 아이디 입력 받기.
*   **Password Stage:** 비밀번호 검증하기.
*   **Authenticator Validation Stage:** TOTP(OTP) 검증하기.
*   **User Write Stage:** DB에 사용자 정보 저장하기.

### Prompt (질문)
사용자에게 무언가 입력을 받아야 할 때 씁니다. (예: 이름, 전화번호, 동의 체크박스)

---

## 2. Policies: The Guard Rails

Flow나 Stage를 실행할지 말지 결정하는 **"조건문"**입니다.
이것이 Authentik의 진짜 힘입니다.

*   **Expression Policy:** Python 코드로 조건을 짭니다. (Week 10에서 심화)
    *   `return request.user.group_attributes['is_vip'] == True`
*   **Reputation Policy:** 수상한 IP나 실패 횟수를 기반으로 차단합니다.
*   **Event Matcher Policy:** 특정 시간대나 이벤트에 반응합니다.

**Binding:**
Stage에 Policy를 붙여서, "이 정책을 통과해야만 이 단계로 넘어간다"를 구현합니다.

---

## 3. Designing a Custom Login Flow

기본 로그인 흐름(`default-authentication-flow`)을 뜯어보면 이렇게 되어 있습니다.

1.  **Identification Stage:** 아이디 입력.
2.  **Password Stage:** 비밀번호 입력.
3.  **MFA Validation Stage:** (MFA 설정된 경우) OTP 입력.
4.  **Login Stage:** 세션 생성 및 리다이렉트.

우리는 이걸 수정해서 **"비밀번호 없는 로그인(Passwordless)"** 흐름을 만들 수도 있습니다.
1.  **Identification Stage:** 아이디 입력.
2.  **WebAuthn Stage:** 지문/FaceID 인증.
3.  **Login Stage:** 끝.

---

## 🛠️ Lab: The "Captcha" Flow

로그인 시도 횟수가 많은 IP에 대해서만 Captcha(로봇 아님 체크)를 띄우는 Flow를 만들어봅니다.

### 1. Stage 생성
*   **System -> Stages -> Create:** `Captcha Stage` 선택.
*   Public Key/Private Key: Google reCAPTCHA나 Turnstile 키 입력.

### 2. Policy 생성
*   **System -> Policies -> Create:** `Reputation Policy` 선택.
*   Condition: 로그인 실패 횟수가 5회 이상일 때.

### 3. Flow 편집
*   **Flows -> default-authentication-flow** 선택 -> **Stage Bindings**.
*   **Identification Stage** **앞에** Captcha Stage를 추가합니다.
*   Captcha Stage의 **Policy Binding**에 아까 만든 Reputation Policy를 연결합니다.
*   **결과:** 평소엔 안 보이다가, 실패를 많이 한 IP에게만 Captcha가 뜹니다.

---

## 📝 6주차 과제: Terms of Service (ToS) Flow

**목표:** 사용자가 최초 로그인할 때만 "이용 약관"에 동의하게 만드세요.

1.  **Prompt 생성:** "약관에 동의합니다" 체크박스 (`terms-agreed`).
2.  **Prompt Stage 생성:** 위 Prompt를 보여주는 스테이지.
3.  **Flow 생성:** `ToS-Flow`를 만들고 Prompt Stage를 넣습니다.
4.  **Policy 생성:** 사용자의 `attributes.tos_agreed`가 `True`가 **아닌** 경우에만 통과하는 Python Policy.
5.  **Main Login Flow 수정:** 로그인 완료 **직전**에 이 `ToS-Flow`를 서브 플로우로 호출하거나, Stage를 끼워 넣고 Policy를 바인딩하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Authentik에서 사용자의 입력을 받는 화면 요소를 무엇이라 부르나요? (Prompt)
2.  **Q2.** 하나의 Flow 안에서 여러 Stage의 실행 순서를 결정하는 것은 무엇인가요? (Stage Binding의 Order 번호)
3.  **Q3.** "평일 업무 시간(9-6)에만 로그인을 허용한다"는 요구사항을 구현하려면 무엇을 사용해야 할까요? (Policy - Time based)

---

이 Flow 엔진을 이해했다면, 이제 Authentik으로 못 할 인증 시나리오는 없습니다.
다음 주에는 이미 다뤘던 Week 7(LDAP)을 넘어, Week 8에서 이 Flow를 **외부 애플리케이션**에 적용하는 방법을 배웁니다.

**Flow like water.**
