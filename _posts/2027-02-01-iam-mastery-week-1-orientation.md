---
layout: post
title: "IAM Mastery with Authentik: Week 1 - Orientation & Identity Fundamentals"
---

안녕하세요, 보안 꿈나무 여러분.

저는 지난 30년간 글로벌 테크 기업의 보안 팀을 이끌며, 수십만 명의 임직원과 고객의 디지털 신원(Identity)을 보호해왔습니다. 90년대의 단순한 패스워드 파일부터, 2000년대의 복잡한 Active Directory 숲(Forest), 그리고 현대의 클라우드 기반 SSO까지, 인증 시스템의 역사는 곧 제 경력의 역사이기도 합니다.

오늘부터 12주간, 우리는 오픈소스 차세대 IAM(Identity and Access Management) 솔루션인 **Authentik**을 사용하여, 기업 수준의 단일 로그인(SSO) 시스템을 바닥부터 구축해볼 것입니다.

이 과정은 단순히 툴 사용법을 익히는 매뉴얼이 아닙니다. **LDAP, OAuth 2.0, OIDC, SAML** 등 현대 웹을 지탱하는 핵심 인증 프로토콜의 원리를 뼈속까지 이해하고, 이를 실무에 적용하는 **IAM 아키텍트** 양성 과정입니다.

---

## 📅 12주 완성 커리큘럼 (Syllabus)

우리의 목표는 명확합니다. 12주 후, 여러분은 "우리 회사 로그인 시스템 좀 만들어줘"라는 요청에 주저 없이 Authentik 인스턴스를 띄우고, Active Directory를 연동하고, Slack과 GitHub 로그인을 붙일 수 있어야 합니다.

### Part 1: Protocols & Foundations (프로토콜과 기초)
*   **W1:** 오리엔테이션 & Identity Fundamentals (AuthN vs AuthZ)
*   **W2:** The Directory: LDAP & Active Directory Deep Dive (구조와 원리)
*   **W3:** The Handshake: OAuth 2.0 & OIDC Deep Dive (모던 인증의 표준)
*   **W4:** The Enterprise: SAML 2.0 Deep Dive (레거시와 엔터프라이즈)

### Part 2: Authentik Implementation (Authentik 구축 및 활용)
*   **W5:** Installation & Architecture (Docker Compose, Geo-IP, Worker)
*   **W6:** The Engine: Flows, Stages, and Policies (Authentik만의 강력함)
*   **W7:** Inbound Sync: Active Directory/LDAP 연동 실습
*   **W8:** Outbound Auth: 애플리케이션 연동 (Proxy Provider vs Forward Auth)

### Part 3: Security & Advanced Engineering (보안 및 고급 엔지니어링)
*   **W9:** Multi-Factor Authentication (MFA) & Passwordless (WebAuthn/FIDO2)
*   **W10:** Policy Engineering (Python 기반의 동적 접근 제어)
*   **W11:** Auditing, Logging & Compliance (누가 언제 무엇을 했는가)
*   **W12:** Capstone: High Availability(HA) 구성 및 운영 전략

---

## 🎓 1주차 강의: Identity Fundamentals

### 1. The "Identity" Crisis
컴퓨터 공학에서 가장 어려운 두 가지가 캐시 무효화와 이름 짓기라고 하죠? 보안에서는 **"이 사람이 진짜 그 사람인가?"**를 증명하는 것이 가장 어렵습니다.

시스템이 커지면 사용자는 수십 개의 아이디와 패스워드를 기억해야 합니다.
*   Gmail 비밀번호
*   회사 인트라넷 비밀번호
*   AWS 콘솔 비밀번호
*   Slack 비밀번호...

이것을 해결하기 위해 등장한 것이 **IAM(Identity and Access Management)**과 **SSO(Single Sign-On)**입니다.

### 2. AuthN vs AuthZ: The Vital Difference
이 둘을 혼동하면 보안 사고가 납니다.

*   **Authentication (인증, AuthN):** "당신 누구세요?" (Who are you?)
    *   예: 신분증 검사, 아이디/비번 입력, 지문 인식.
    *   결과: "이 사람은 '홍길동'이 맞습니다."
*   **Authorization (인가, AuthZ):** "당신 뭘 할 수 있나요?" (What can you do?)
    *   예: 영화표 검사(19세 관람가), 출입증(연구소 출입 불가).
    *   결과: "이 사람은 '관리자 페이지'에 접근할 권한이 있습니다."

**Authentik**은 이 두 가지를 모두 처리하는 **IdP (Identity Provider)**입니다.

### 3. The Players in IAM
앞으로 계속 나올 용어들입니다. 지금 확실히 정리합시다.

*   **User (Principal):** 로그인을 시도하는 사람(또는 봇).
*   **IdP (Identity Provider):** 신분증 발급처. 사용자 정보를 가지고 있고, 인증을 수행합니다. (예: Authentik, Okta, Google)
*   **SP (Service Provider) / RP (Relying Party):** 서비스를 제공하는 앱. 인증을 IdP에게 위임합니다. (예: Jenkins, Grafana, Slack)

**비유:**
여러분이 클럽(SP)에 가려고 합니다. 기도(SP의 로그인 모듈)가 막습니다.
"신분증 보여주세요."
여러분이 주민등록증(Token)을 보여줍니다. 이 주민등록증은 대한민국 정부(IdP)가 발급했습니다.
기도는 여러분 얼굴과 주민등록증 사진을 대조(AuthN)하고, 나이를 확인(AuthZ)한 뒤 들여보내줍니다.

---

## 🛠️ Lab: Concept Check

Authentik을 설치하기 전에, 개념부터 잡읍시다.

### 시나리오
회사의 신입 사원 'Alice'가 입사했습니다.
1.  인사팀은 Alice의 정보를 `HR 시스템`에 등록합니다.
2.  Alice는 `사내 메신저`에 로그인하려고 합니다.
3.  `사내 메신저`는 비밀번호를 묻지 않고, `SSO 페이지`로 리다이렉트합니다.

**질문:**
*   여기서 IdP는 무엇이 되어야 할까요?
*   HR 시스템과 IdP는 어떻게 연결되어야 할까요? (힌트: 다음 주 주제인 LDAP)

---

## 📝 1주차 과제: Environment Setup

다음 주부터 바로 실습에 들어갑니다. 아래 환경을 준비해 오세요.

1.  **Docker & Docker Compose:** 최신 버전 설치.
2.  **Domain:** 로컬 테스트를 위해 `/etc/hosts`에 `authentik.local.test` 등을 등록하거나, 실제 도메인 준비.
3.  **RAM:** 최소 4GB 이상의 여유 메모리 (Authentik과 테스트용 앱들을 띄워야 합니다).

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 사용자가 로그인을 성공했지만, "권한이 없습니다" 페이지가 떴습니다. 이는 AuthN 실패인가요, AuthZ 실패인가요? (AuthZ)
2.  **Q2.** SSO(Single Sign-On)를 도입했을 때의 보안상 장점과 단점(Risk)은 무엇일까요? (장점: 중앙 관리, 단점: 하나의 키로 모든 문을 염 - Single Point of Failure)
3.  **Q3.** 여러분이 만드는 웹사이트에서 "Google로 로그인" 버튼을 달았습니다. 이때 Google의 역할은? (IdP)

---

다음 주, 우리는 인터넷의 전화번호부, **LDAP**과 **Active Directory**의 숲으로 들어갑니다. IAM의 뿌리를 찾아서 말이죠.

**Secure your identity.**
