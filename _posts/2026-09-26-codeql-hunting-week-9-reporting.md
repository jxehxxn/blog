---
layout: post
title: "Vulnerability Hunting with CodeQL: Week 9 - The Art of the Report (PoC & Impact Analysis)"
---

CodeQL이 "여기 SQL Injection 있어요"라고 알려줬습니다.
그렇다고 바로 개발자에게 메일 보내면 안 됩니다.
**"진짜로?"**라는 질문에 답해야 합니다.

CodeQL은 논리적 가능성을 보여줄 뿐, 실제 공격 가능성을 보장하진 않습니다.
이것을 검증하는 과정이 **Triage(트리아지)**입니다.

---

## 🕵️ Verification (Manual Triage)

CodeQL이 보여준 Path(경로)를 따라가며 코드를 읽으세요.

1.  **Source:** `getParam("id")` -> 진짜 외부에서 호출 가능한가? (URL 매핑 확인)
2.  **Data Flow:** 중간에 `if (id.matches("[0-9]+"))` 같은 검증 로직이 있는데 CodeQL이 놓친 건 아닌가?
3.  **Sink:** `executeQuery(sql)` -> 진짜 위험한가?

### PoC (Proof of Concept)
가장 확실한 증명은 **재현(Reproduction)**입니다.
로컬에 서버를 띄우고, `curl`로 악성 페이로드를 날려보세요. 에러가 터지거나 데이터가 털리면 100%입니다.
(만약 환경 구축이 어렵다면, 코드 흐름만으로 논리적 증명을 해야 합니다.)

---

## 💥 Impact Analysis (CVSS)

취약점이라고 다 같은 게 아닙니다.
*   **Critical (9.0+):** 인증 없이 원격 코드 실행 (Log4Shell)
*   **High (7.0+):** SQL Injection, 중요 정보 유출
*   **Medium:** XSS (보통), CSRF
*   **Low:** 단순 정보 노출 (버전 정보 등)

**CVSS (Common Vulnerability Scoring System)** 계산기를 두드려보며 점수를 매겨보세요. 보고서에 점수가 있으면 전문성이 올라갑니다.

---

## 📝 The Report Structure

좋은 보고서는 개발자를 설득합니다.

1.  **Summary:** 한 줄 요약. (e.g., "SQL Injection in User Search functionality")
2.  **Description:** 상세 설명. CodeQL이 찾은 데이터 흐름을 첨부.
3.  **Impact:** 이 취약점으로 해커가 무엇을 할 수 있는지. (e.g., "Extract all user passwords")
4.  **PoC:** 재현 방법. (`curl` 명령어 등)
5.  **Remediation:** 고치는 법. (e.g., "Use PreparedStatement instead of String concatenation")

---

## 🛠️ Lab: Write it up

지난주 필터링 결과 중 하나를 골라(혹은 가상의 취약점을 설정해) **Markdown 리포트**를 작성해 보세요.
"내가 찾았으니 너네 코드 고쳐라"라는 태도가 아니라, **"당신의 프로젝트를 안전하게 만들기 위해 돕고 싶습니다"**라는 톤앤매너(Tone & Manner)를 유지하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 취약점의 심각도를 점수(0~10)로 정량화하여 평가하는 국제 표준 시스템은?
2.  **Q2.** 발견된 취약점이 실제로 작동함을 입증하기 위해 작성하는 코드나 스크립트를 무엇이라 하나요? (P... o... C...)
3.  **Q3.** CodeQL 결과 중 실제로는 취약점이 아닌데 취약점이라고 보고된 경우를 무엇이라 하나요? (False Positive)

---

다음 주는 **AI-Augmented Triage**입니다.
트리아지와 보고서 작성... 귀찮죠?
LLM(ChatGPT, Claude)에게 이 일을 시키는 방법을 알아봅니다.

**Prove it.**
