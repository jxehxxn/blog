---
layout: post
title: "Vulnerability Hunting with CodeQL: Week 2 - Thinking in Graphs (QL Syntax & Logic)"
---

지난주에 CodeQL 환경을 구축했습니다. 이제 쿼리를 짤 차례입니다.
하지만 QL은 Java나 Python과는 다릅니다. **Datalog** 기반의 논리형 언어입니다.
"어떻게 찾을지(How)"가 아니라 **"무엇을 찾을지(What)"**를 정의해야 합니다.

---

## 🧠 Paradigm Shift (패러다임의 전환)

*   **Imperative (Python):** `for file in files: if "password" in file: print(file)`
*   **Declarative (QL):** "나는 파일 이름에 'password'가 들어가는 파일들의 집합을 원해."

QL은 **집합(Set)**과 **관계(Relation)**를 다룹니다.

---

## 🏗️ Core Syntax

가장 기본적인 구조는 SQL과 비슷해 보이는 `from-where-select` 절입니다.

```ql
import java

from MethodCall call, Method target     // [Variable Declarations]
where call.getTarget() = target         // [Logical Formulas]
  and target.hasName("exec")            // [Conditions]
select call, "Suspicious execution"     // [Output]
```

### 1. Predicates (서술어)
재사용 가능한 논리 블록입니다. 함수처럼 보이지만, 사실은 "참/거짓"을 판별하는 논리식입니다.

```ql
predicate isPublic(Method m) {
  m.isPublic()
}
```

### 2. Classes (클래스)
QL의 클래스는 "객체 생성"을 위한 게 아닙니다. **"기존 집합의 부분집합"**을 정의하는 것입니다.

```ql
class PublicMethod extends Method {
  // Method 중에서 isPublic()이 참인 것들만 모은 집합
  PublicMethod() { this.isPublic() }
}
```

---

## 🛠️ Lab: Finding Bad Patterns

샘플 DB(Java)를 대상으로 위험한 함수 호출을 찾아봅시다.

### Task: `Runtime.exec()` 찾기
자바에서 외부 명령어를 실행하는 `Runtime.exec`는 대표적인 **Command Injection** 후보입니다.

```ql
import java

from MethodCall call
where call.getTarget().hasName("exec")
  and call.getTarget().getDeclaringType().hasQualifiedName("java.lang", "Runtime")
select call, "Found Runtime.exec() call!"
```
이 쿼리를 실행하면 코드 내의 모든 `exec` 호출 지점이 뜹니다.

---

## 📝 2주차 과제: Code Smell 찾기

보안 취약점은 아니지만, 유지보수에 나쁜 "Code Smell"을 찾아보세요.
**"파라미터가 5개 이상인 모든 메소드"**를 찾는 쿼리를 작성하세요.

**힌트:**
`Method` 클래스에는 `getNumberOfParameters()`라는 프레디케이트가 있습니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** QL 언어는 명령형(Imperative) 언어인가요, 선언형(Declarative) 언어인가요?
2.  **Q2.** QL에서 재사용 가능한 논리 조건을 정의하는 키워드는 무엇인가요? (P...)
3.  **Q3.** `where` 절에서 두 조건이 모두 참이어야 함을 나타내는 논리 연산자는? (`and` / `or`)

---

다음 주는 CodeQL의 꽃, **Data Flow Analysis(데이터 흐름 분석)**입니다.
단순히 "위험한 함수를 썼다"고 해서 취약점이 아닙니다.
**"해커가 입력한 데이터"**가 그 함수로 흘러들어가야 취약점입니다.
그 연결고리를 찾는 **Taint Tracking**을 배웁니다.

**Think logically.**
