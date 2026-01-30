---
layout: post
title: "Vulnerability Hunting with CodeQL: Week 12 - Capstone Project (The Great Hunt)"
---

여기까지 오신 여러분, 고생 많으셨습니다.
우리는 CodeQL의 기초 문법부터 시작해, 데이터 흐름 분석, 자동화 파이프라인, AI 트리아지, 그리고 윤리적 제보까지 모든 과정을 밟았습니다.
이제 연습은 끝났습니다. **실전(The Great Hunt)**입니다.

---

## 🏆 The Capstone Project

여러분의 임무는 하나입니다.
**"오픈소스 프로젝트에서 유효한 취약점(또는 유의미한 Code Smell)을 찾아내고 리포트하라."**

### 🗺️ Mission Roadmap

1.  **Target Selection (W5):** GitHub API를 돌려 타겟 100개를 확보하세요. (Java, Python, JS 등 본인 주력 언어)
2.  **Build Pipeline (W6):** 자동화 스크립트로 100개의 DB를 생성하세요.
3.  **Scan (W2~W4):** 표준 쿼리(`security-extended`)와 여러분이 짠 커스텀 쿼리(W7)를 돌리세요.
4.  **Triage (W8, W10):** 필터링 스크립트와 AI를 동원해 FP를 걸러내고, 후보군을 추리세요.
5.  **Verify (W9):** 살아남은 후보군을 직접 눈으로 확인하고(Manual), 가능하다면 로컬에서 PoC를 돌리세요.
6.  **Report (W11):** 진짜라면 메인테이너에게 제보하고, 아니라면(FP라면) 왜 CodeQL이 오탐을 냈는지 분석 보고서를 쓰세요.

---

## 🎁 Deliverables (제출물)

이 프로젝트의 결과물은 여러분의 **포트폴리오**가 될 것입니다.
다음 내용을 포함하여 GitHub 블로그나 저장소에 정리하세요.

1.  **The CodeQL Query:** 여러분이 사용한 쿼리 (표준 쿼리 수정본 or 커스텀 쿼리).
2.  **The Analysis:** 발견된 취약점의 데이터 흐름도 (CodeQL Path).
3.  **The Verdict:** 이것이 진짜 취약점인지, 오탐인지에 대한 결론과 그 이유.
4.  **The Automation:** 사용한 파이썬 스크립트들 (Targeting, Pipeline, AI Filter).

---

## 🚀 The Future: Variant Analysis

운 좋게 취약점 하나를 찾았나요? 거기서 멈추지 마세요.
그 취약점 패턴을 **일반화된 쿼리**로 만드세요.
그리고 전 세계 모든 오픈소스를 대상으로 그 쿼리를 돌리세요.
이것이 **Variant Analysis**입니다. 하나를 찾으면 100개를 찾을 수 있습니다.
GitHub Security Lab이 하는 일이 바로 이것입니다.

---

## 👋 Closing

CodeQL은 배우기 어렵습니다. 러닝 커브가 절벽 같습니다.
하지만 그 절벽을 기어오른 사람에게만 보이는 풍경이 있습니다.
남들은 수동으로 며칠 걸려 찾을 버그를, 여러분은 쿼리 한 줄로 수천 개 프로젝트에서 찾아낼 수 있습니다.

이제 여러분은 **Security Researcher**입니다.
CodeQL 데이터베이스는 여러분의 바다입니다. 마음껏 낚시를 즐기세요.

**Go fish.**

---
*Professor. Antigravity*
