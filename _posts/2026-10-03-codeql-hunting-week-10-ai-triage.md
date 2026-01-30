---
layout: post
title: "Vulnerability Hunting with CodeQL: Week 10 - Cyborg Hunting (AI-Augmented Triage)"
---

50개의 잠재적 취약점을 사람이 일일이 코드를 읽고 분석하려면 며칠이 걸립니다.
하지만 우리에겐 **LLM (Large Language Model)**이라는 훌륭한 조수가 있습니다.

CodeQL의 정밀함(논리) + LLM의 이해력(문맥) = **Cyborg Hunter**.
이번 주는 이 조합으로 생산성을 10배 높이는 법을 배웁니다.

---

## 🤖 The Prompt Engineering for Security

LLM에게 무작정 "이거 취약해?"라고 물으면 헛소리(Hallucination)를 합니다.
CodeQL의 결과를 바탕으로 구체적인 **Context**를 줘야 합니다.

### Input
*   **Source Code Snippet:** 취약점이 의심되는 함수 코드.
*   **CodeQL Path:** Source에서 Sink로 이어지는 경로 정보.
*   **Query Description:** 이 쿼리가 무엇을 찾는 것인지(예: SQL Injection).

### Prompt Example
```text
You are a Security Expert. Analyzing a CodeQL result.
Here is the data flow path:
1. Source: request.getParameter("id") in UserServlet.java:20
2. Sink: stmt.execute("SELECT * FROM users WHERE id=" + id) in Dao.java:45

Here is the code snippet for UserServlet.java:
... (code) ...

Here is the code snippet for Dao.java:
... (code) ...

Task:
1. Verify if the 'id' variable is sanitized between line 20 and 45.
2. If not sanitized, explain how to exploit it.
3. If it is a False Positive, explain why.
```

---

## 🛠️ Lab: The AI Assistant Script

8주차에 만든 `sarif_filter.py`를 업그레이드합니다.
OpenAI API (또는 Anthropic API)를 연동합니다.

1.  SARIF에서 `ruleId`와 `locations`를 파싱합니다.
2.  해당 파일의 소스 코드를 읽어옵니다. (Git Clone 해둔 폴더에서)
3.  LLM API에 프롬프트를 보냅니다.
4.  LLM의 답변(True Positive / False Positive)을 CSV에 추가합니다.

이제 여러분은 커피 한 잔 마시는 동안, AI가 1차 필터링을 끝내놓을 것입니다.
**"AI가 High Confidence로 찍은 것"**부터 먼저 보면 됩니다.

---

## 📝 10주차 과제: Fix Suggestion

LLM에게 취약점 판별뿐만 아니라, **"수정 코드(Fix)"**를 제안해달라고 요청하는 프롬프트를 작성해보세요.
나중에 리포트 쓸 때 `Remediation` 섹션에 그대로 붙여넣을 수 있습니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** LLM이 사실이 아닌 내용을 그럴듯하게 지어내서 답변하는 현상을 보안 분야에서 주의해야 하는데, 이 현상의 이름은? (H...)
2.  **Q2.** LLM에게 보안 분석을 맡길 때, 정확도를 높이기 위해 제공해야 하는 핵심 정보 두 가지는? (코드 스니펫, 데이터 흐름 경로 등)
3.  **Q3.** CodeQL의 구조적 분석 능력과 LLM의 자연어 이해 능력을 결합하여 분석 효율을 높이는 접근 방식을 이 강의에서 무엇이라 비유했나요? (Cyborg ...)

---

다음 주는 **Responsible Disclosure**입니다.
취약점을 찾았습니다. AI도 진짜라고 합니다.
이제 이걸 세상에 알려야 하는데... 트위터에 올리면 고소당합니다.
안전하고 윤리적으로 제보하고, 내 이름으로 된 **CVE**를 따내는 절차를 배웁니다.

**Automate the mind.**
