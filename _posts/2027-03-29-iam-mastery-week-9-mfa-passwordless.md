---
layout: post
title: "IAM Mastery with Authentik: Week 9 - MFA & Passwordless Security"
---

비밀번호는 죽었습니다(Password is dead).
길고 복잡하게 만들어봤자 피싱 사이트 한 번 잘못 들어가면 끝입니다. 매번 바꾸라고 하면 `Summer2024!`, `Winter2024!` 처럼 예측 가능한 패턴만 씁니다.

이제 보안의 표준은 **MFA(Multi-Factor Authentication)**, 더 나아가 **Passwordless**입니다.
Authentik은 이 분야에서 상용 솔루션(Okta, Duo) 뺨치는 기능을 제공합니다.

---

## 1. MFA Types & Authentik Support

MFA는 "지식(비번) + 소유(폰) + 생체(지문)" 중 2가지를 섞는 것입니다.

### Authentik이 지원하는 MFA
1.  **TOTP (Time-based OTP):** Google Authenticator. 가장 기본.
2.  **WebAuthn (FIDO2):** YubiKey, Windows Hello, TouchID. **가장 강력함.**
3.  **Duo:** Cisco Duo 연동.
4.  **Email/SMS:** 코드를 보내서 확인. (SMS는 별도 게이트웨이 필요)

---

## 2. Setting up MFA Flow

Week 6에서 배운 Flow를 써먹을 때입니다.
기본적으로 Authentik은 사용자가 MFA를 등록했으면 로그인 시 물어보고, 안 했으면 넘어갑니다.

하지만 **"강제(Enforcement)"**하고 싶다면?

### MFA Setup Stage
사용자가 MFA를 등록하는 단계입니다.
`default-authenticator-totp-setup` 등의 Flow가 이미 내장되어 있습니다.

### MFA Validation Stage
로그인 시 검사하는 단계입니다.
`default-authentication-flow` 안에 `default-authentication-mfa-validation` 스테이지가 있습니다.

**정책(Policy)으로 강제하기:**
1.  **Policy 생성:** `MFA Required Policy`. "사용자가 MFA를 설정하지 않았으면 False 반환".
2.  **Flow 수정:** 로그인 Flow 중간에 "MFA 설정 Flow"로 강제로 보내버리는 스테이지를 추가하고, 위 정책을 바인딩합니다.

---

## 3. Passwordless: The Future

비밀번호 입력 창 자체를 없애버리는 것입니다.
사용자는 아이디만 입력하고, 스마트폰의 지문 센서에 손을 대면 로그인이 끝납니다. (Passkeys)

### 구현 방법
1.  **Identification Stage** 설정에서 `Passwordless` 모드를 켭니다.
2.  사용자는 미리 **WebAuthn(Passkey)** 기기를 등록해둬야 합니다.
3.  로그인 시 Authentik은 비밀번호를 묻지 않고 브라우저에게 "WebAuthn 인증해줘"라고 요청합니다.

이 방식은 **피싱(Phishing)이 불가능**합니다. 가짜 사이트는 내 진짜 도메인의 WebAuthn 서명을 받을 수 없기 때문입니다.

---

## 🛠️ Lab: Configuring TOTP & WebAuthn

여러분의 계정에 강력한 보안을 걸어봅시다.

1.  **사용자 설정:** 우측 상단 프로필 -> Settings.
2.  **TOTP 등록:** Google Authenticator 앱으로 QR 코드를 찍고 등록합니다.
3.  **WebAuthn 등록:** (노트북에 지문인식이나 맥북 TouchID가 있다면) Passkey를 등록합니다.
4.  **로그인 테스트:**
    *   시크릿 모드 접속.
    *   아이디/비번 입력 후.
    *   OTP 입력 창이 뜨는지 확인.
    *   또는 "Use Security Key" 버튼을 눌러 지문으로 로그인되는지 확인.

---

## 📝 9주차 과제: MFA Enforcement Policy

**목표:** `Admins` 그룹에 속한 사용자는 **반드시** MFA를 설정해야만 로그인할 수 있도록 강제하는 정책을 구현하세요.

1.  **Expression Policy**를 만듭니다.
    ```python
    # 사용자가 Admins 그룹이고, MFA 장비가 하나도 없으면
    is_admin = "Admins" in request.user.group_names
    has_mfa = len(request.user.mfa_devices) > 0
    
    if is_admin and not has_mfa:
        return True # 이 경우에만 실행 (MFA 설정 페이지로 납치)
    return False
    ```
2.  로그인 Flow에 **"MFA Setup Stage"**를 추가하고, 이 정책을 바인딩합니다.
3.  Admin 계정의 MFA를 모두 지우고 로그인 시도 시, 설정 화면으로 강제 이동되는지 확인하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** MFA 방식 중 피싱(Phishing) 공격에 대해 가장 내성이 강한(안전한) 방식은? (WebAuthn / FIDO2 / Passkeys)
2.  **Q2.** Authentik에서 사용자가 자신의 MFA 디바이스를 관리할 수 있는 기본 URL 경로는? (`/if/user/#/settings`)
3.  **Q3.** TOTP 알고리즘은 서버와 클라이언트 간에 무엇이 일치해야 정상 동작하나요? (시간 - Time)

---

보안을 강화했으니, 이제 더 세밀한 제어가 필요합니다.
다음 주, Python 코드로 Authentik을 마음대로 주무르는 **Advanced Policy Engineering**을 배웁니다. 코딩할 준비 하세요.

**Trust nothing, verify everything.**
