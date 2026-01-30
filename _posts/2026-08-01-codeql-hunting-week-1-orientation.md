---
layout: post
title: "Vulnerability Hunting with CodeQL: Week 1 - Orientation & Architecture"
---

환영합니다, 예비 보안 연구원 여러분.

저는 지난 30년간 보안 업계에 몸담으며, GitHub Security Lab에서 CodeQL을 이용해 수많은 오픈소스의 0-day 취약점을 찾아내고 제보해온 여러분의 멘토입니다. 오늘부터 12주간, 우리는 단순한 취약점 스캐너 돌리기가 아닌, **코드를 데이터처럼 쿼리하여(Code as Data)** 숨겨진 논리적 오류와 취약점을 찾아내는 **CodeQL**의 세계로 들어갑니다.

우리의 목표는 명확합니다. 12주 후, 여러분은 실제 오픈소스 프로젝트에서 **CVE(Common Vulnerabilities and Exposures)**를 발급받을 수 있는 수준의 실력을 갖추게 될 것입니다.

---

## 📅 12주 완성 커리큘럼 (Syllabus)

CodeQL은 배우기 어렵습니다. 하지만 강력합니다. 우리는 기초 문법부터 시작해, 나중에는 AI를 활용한 자동화 파이프라인까지 구축할 것입니다.

### Part 1: Foundations (CodeQL의 기초)
*   **W1:** 오리엔테이션 & CodeQL 아키텍처 (Database, QL, CLI vs VSCode)
*   **W2:** QL 문법 기초 (Predicates, Classes, Select)
*   **W3:** 데이터 흐름 분석 (Taint Tracking, Source, Sink, Sanitizer)
*   **W4:** 제어 흐름 & AST 분석 (Basic Blocks, Guards)

### Part 2: Automation & Hunting (자동화와 사냥)
*   **W5:** 타겟 선정 전략 (GHSA API, Project Scoring, Automation)
*   **W6:** 파이프라인 구축 (Auto-clone, Auto-DB creation)
*   **W7:** 커스텀 쿼리 작성 (새로운 프레임워크 모델링)
*   **W8:** 대규모 결과 분석 (SARIF 파싱, 오탐 필터링)

### Part 3: Reporting & AI Augmentation (보고 및 AI 활용)
*   **W9:** 리포트 작성의 기술 (PoC, Root Cause, Impact)
*   **W10:** AI 기반 Triage (LLM을 이용한 검증 및 리포트 초안 작성)
*   **W11:** 윤리적 제보 (Responsible Disclosure, CNA, Communication)
*   **W12:** 캡스톤 프로젝트 (실전 헌팅, 리포팅, CVE 도전)

---

## 🎓 1주차 강의: What is CodeQL?

### 1. Code as Data (코드가 곧 데이터다)
보통 우리는 코드를 텍스트 파일로 봅니다. `grep`으로 `exec()`를 찾죠.
하지만 CodeQL은 코드를 **계층적 데이터베이스**로 변환합니다.
*   클래스, 메소드, 변수, 호출 관계... 이 모든 것이 DB 테이블의 행(Row)이 됩니다.

### 2. SQL for Code
SQL이 데이터베이스에서 데이터를 조회하듯, QL(Query Language)은 코드베이스에서 패턴을 조회합니다.

*   **SQL:** `SELECT * FROM users WHERE age > 18`
*   **QL:** `from MethodCall call where call.getMethod().hasName("exec") select call`

이러면 단순히 텍스트 매칭이 아니라, **"exec라는 이름의 메소드를 호출하는 모든 지점"**을 정확히 찾아냅니다.

### 3. Architecture
CodeQL은 크게 3단계로 작동합니다.
1.  **Extractor:** 소스 코드를 빌드하면서 정보를 추출합니다. (Java, C++ 등 컴파일 언어는 빌드 과정 필수)
2.  **Database:** 추출된 정보를 관계형 데이터로 저장합니다. (스냅샷)
3.  **Query Server:** 우리가 작성한 `.ql` 쿼리를 실행하여 결과를 반환합니다.

---

## 🛠️ Lab: Setup (환경 구축)

CodeQL을 시작하려면 준비물이 꽤 필요합니다. 차근차근 따라오세요.

### Step 1: VSCode & Extension
1.  Visual Studio Code를 설치합니다.
2.  마켓플레이스에서 **"CodeQL"** 확장 프로그램을 설치합니다.

### Step 2: CodeQL CLI (자동화를 위해 필수)
1.  GitHub Releases에서 `codeql-bundle`을 다운로드합니다.
2.  압축을 풀고 PATH에 등록합니다. (`codeql --version`으로 확인)

### Step 3: Starter Workspace
쿼리를 작성할 작업 공간을 가져옵니다.
```bash
git clone https://github.com/github/vscode-codeql-starter.git
cd vscode-codeql-starter
git submodule update --init --remote
```
VSCode에서 이 폴더를 엽니다. (`code .`)

### Step 4: Sample Database
연습용 데이터베이스가 필요합니다. GitHub에서 아무 오픈소스 프로젝트나 가져와서 DB를 만들어도 되지만, 처음엔 가벼운 걸 추천합니다.
(VSCode 확장에서 "Download Database from GitHub" 기능을 써서 `apache/commons-lang` 같은 걸 받아보세요.)

---

## 📝 1주차 과제: Hello World Query

VSCode에서 `codeql-custom-queries-java` (언어는 선택) 폴더 안에 `hello.ql` 파일을 만들고 실행해 보세요.

```ql
import java

from Method m
select m, "Hello World! Found a method."
```
데이터베이스 안의 모든 메소드를 찾아서 출력할 것입니다.
이것이 여러분의 첫 번째 CodeQL 쿼리입니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** CodeQL이 소스 코드를 분석하기 위해 가장 먼저 생성해야 하는 중간 결과물은 무엇인가요? (D...)
2.  **Q2.** CodeQL은 코드를 텍스트가 아닌 무엇으로 취급하여 쿼리하나요? (Code as ...)
3.  **Q3.** 컴파일 언어(Java, C++)의 CodeQL 데이터베이스를 생성할 때 반드시 수행되어야 하는 과정은? (B...)

---

다음 주에는 **QL 문법**을 본격적으로 배웁니다.
"객체지향"이 아니라 **"논리 프로그래밍(Logic Programming)"**이라는 낯선 패러다임에 적응해야 합니다.
머리를 말랑말랑하게 준비해 오세요.

**Happy Hunting.**
