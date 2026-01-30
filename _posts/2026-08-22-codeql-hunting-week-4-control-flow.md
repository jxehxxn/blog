---
layout: post
title: "Vulnerability Hunting with CodeQL: Week 4 - The Structure of Chaos (Control Flow & AST Analysis)"
---

Taint Tracking은 강력하지만 맹점이 있습니다. "흐름"만 볼 뿐 "조건"을 잘 못 봅니다.
예를 들어, 관리자 권한을 체크하는 `if`문 안에서 실행되는 위험한 코드는(논리적으로는) 안전할 수 있습니다.
이런 문맥을 이해하려면 코드의 구조(**AST**)와 흐름(**CFG**)을 분석해야 합니다.

---

## 🌳 AST (Abstract Syntax Tree)

코드를 트리 구조로 봅니다.
*   `Class` -> `Method` -> `Block` -> `IfStmt` -> `Block` -> `Expr`

이것을 이용하면 "모든 `catch` 블록을 찾되, 내용이 비어있는(Empty) 것만 찾아라" 같은 쿼리가 가능합니다.

```ql
from CatchClause cc
where cc.getBlock().getNumStmt() = 0 // 내용물이 0개인 블록
select cc, "Empty catch block - swallowing exceptions!"
```

---

## 🔀 CFG (Control Flow Graph)

코드의 실행 순서를 그래프로 봅니다.
**Basic Block(기본 블록)**들이 화살표로 연결됩니다.
*   `getASuccessor()`: 다음 실행될 블록
*   `getAPredecessor()`: 이전에 실행된 블록

### Guards (가드)
보안에서 가장 중요한 개념입니다. "이 변수가 null이 아닌지 검사했나?", "이 유저가 admin인지 검사했나?"
CodeQL에는 `Guards` 라이브러리가 있어 이를 쉽게 체크할 수 있습니다.

```ql
import semmle.code.java.controlflow.Guards

from Guard guard, Expr e
where guard.controls(e.getBasicBlock(), true) // guard가 참일 때만 e가 실행됨
  and guard.(MethodCall).getMethod().hasName("isAuthenticated")
select e, "This expression is protected by authentication!"
```

---

## 🛠️ Lab: Guarded Logic

"권한 체크 없이 중요한 함수(`deleteUser`)를 호출하는 곳"을 찾아봅시다.

1.  Sink: `deleteUser()` 호출.
2.  Guard: `isAdmin()` 체크.
3.  쿼리 로직: Sink를 찾고, 그 Sink를 제어(Control)하는 Guard가 `isAdmin`인지 확인. 만약 제어하는 Guard가 없다면? **Broken Access Control** 취약점입니다.

---

## 📝 4주차 과제: Dead Code 찾기

Control Flow를 이용해 **"항상 거짓인 if문"**을 찾아보세요.
조건식(Condition)이 상수 `false`이거나, 논리적으로 절대 도달할 수 없는 블록을 찾는 것입니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 소스 코드의 문법적 구조를 트리 형태로 표현한 것을 약어로 무엇이라 하나요?
2.  **Q2.** 프로그램의 실행 경로(순서)를 그래프로 표현한 것을 약어로 무엇이라 하나요?
3.  **Q3.** 특정 코드 블록이 실행되기 위해 반드시 만족해야 하는 조건(예: if문, null 체크)을 CodeQL 용어로 무엇이라 하나요? (G...)

---

기초(Part 1)가 끝났습니다. 이제 문법은 다 배웠습니다.
다음 주부터는 **Part 2: Automation & Hunting**입니다.
실제 오픈소스 프로젝트를 **선정(Targeting)**하고, 자동으로 **스캔(Scanning)**하는 파이프라인을 구축할 것입니다.

**Structure is logic.**
