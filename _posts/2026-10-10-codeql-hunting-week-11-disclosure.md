---
layout: post
title: "Vulnerability Hunting with CodeQL: Week 11 - Ethics & Protocol (Responsible Disclosure & CVEs)"
---

취약점 헌팅의 마지막 단계는 기술이 아니라 **커뮤니케이션**입니다.
여러분이 찾은 0-day는 무기입니다. 잘못 다루면 사람을 다치게 하거나(악용), 여러분이 다칩니다(법적 문제).
우리는 **White Hat**으로서 책임감 있는 제보(Responsible Disclosure) 절차를 따라야 합니다.

---

## 📜 The Golden Rule

**"Do No Harm."**
*   절대 실제 운영 중인 서비스(Live Service)를 공격하지 마세요. (우리는 로컬에 띄워서 테스트했습니다.)
*   패치가 나오기 전까지 절대 대중에게 공개하지 마세요.

---

## 📬 Channels: 어디에 제보하나?

1.  **GitHub Security Advisories:** 가장 추천하는 방법입니다. 해당 레포지토리의 `Security` 탭에서 `Report a vulnerability` 버튼을 누르면 **Private Fork**가 생성되어 메인테이너와 비공개로 대화할 수 있습니다.
2.  **Security.md / Security.txt:** 프로젝트 루트에 보안 정책 파일이 있는지 보세요. 보통 `security@domain.com` 같은 이메일이 적혀 있습니다.
3.  **CNA (CVE Numbering Authority):** 메인테이너가 잠적했거나 연락이 안 되면, MITRE나 GitHub 같은 CNA에 직접 연락해서 CVE를 요청할 수도 있습니다.

---

## ⏳ The Timeline (국룰)

1.  **Day 0:** 제보 발송. (암호화된 채널 권장)
2.  **Day 1~7:** 메인테이너 응답 (Triage & Acknowledgement).
3.  **Day 30~60:** 패치 개발 및 검증.
4.  **Day 90:** **Public Disclosure.** (패치가 배포된 후 공개)
    *   구글 Project Zero가 정립한 표준입니다. 90일이 지나도 패치를 안 하면 공개해버린다고 경고합니다. (사용자 보호를 위해)

---

## 🛠️ Lab: The Simulation

가상의 시나리오로 이메일을 작성해 봅시다.

> **Subject:** [Security Report] SQL Injection Vulnerability in User Search
>
> **Body:**
> 안녕하세요, 보안 연구원 OOO입니다.
> 귀하의 프로젝트(XYZ)에서 심각한 SQL Injection 취약점을 발견하여 제보 드립니다.
>
> **요약:** 사용자 검색 기능에서 입력값 검증 부재로 인해 DB 탈취가 가능합니다.
> **상세:** (CodeQL 리포트 및 PoC 첨부)
> **파급력:** 공격자가 관리자 계정을 탈취할 수 있습니다.
>
> 사용자의 안전을 위해 90일간 비공개(Embargo)를 유지하겠습니다.
> 확인 부탁드립니다.

이런 템플릿을 미리 만들어두세요.

---

## 📝 11주차 과제: Security Policy 찾기

유명 오픈소스 프로젝트(예: Kubernetes, React, TensorFlow)의 보안 정책을 찾아보세요.
CVE 발급 절차나 제보 포상금(Bounty)에 대한 내용이 있는지 확인해 보세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 발견된 취약점을 대중에게 공개하기 전에 개발자에게 먼저 알리고 패치할 시간을 주는 윤리적 제보 방식을 무엇이라 하나요?
2.  **Q2.** 취약점에 고유 번호(ID)를 부여하고 관리하는 국제적인 식별 시스템의 이름은? (C...)
3.  **Q3.** 일반적으로 통용되는 취약점 제보 후 공개까지의 유예 기간(Embargo)은 며칠인가요? (Google Project Zero 기준)

---

다음 주는 대망의 **마지막, Capstone Project**입니다.
지금까지 갈고닦은 모든 무기(CodeQL, Pipeline, AI, Report)를 들고
진짜 오픈소스의 바다로 나갑니다.
여러분의 이름이 박힌 CVE를 따러 갑시다.

**Respect the process.**
