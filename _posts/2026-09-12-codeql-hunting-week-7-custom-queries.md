---
layout: post
title: "Vulnerability Hunting with CodeQL: Week 7 - Zero-Day Hunter (Writing Custom Queries)"
---

기본 쿼리 스위트(`security-extended.qls`)는 훌륭하지만, 한계가 있습니다.
널리 알려진 프레임워크(Spring, Flask)만 지원하죠.
만약 타겟이 **듣보잡 프레임워크**나 **자체 제작한 라이브러리**를 쓴다면? CodeQL은 Source도 Sink도 인식하지 못합니다.

이때가 바로 **Custom Query**가 빛을 발하는 순간입니다.

---

## 🧩 Modeling a Library

시나리오: `FastWeb`이라는 새로운 자바 웹 프레임워크가 있습니다.
이 프레임워크는 `FastWeb.getParam("id")`으로 사용자 입력을 받습니다.
CodeQL은 이게 Source인지 모릅니다. 우리가 알려줘야 합니다.

### 1. Identify the Source
소스 코드를 보니 `com.fastweb.Request` 클래스의 `getParam` 메소드네요.

### 2. Extend the Class
QL에서 이 메소드를 정의합니다.

```ql
import java
import semmle.code.java.dataflow.FlowSources

class FastWebSource extends RemoteUserInput {
  FastWebSource() {
    exists(MethodCall ma |
      ma.getMethod().hasName("getParam") and
      ma.getMethod().getDeclaringType().hasName("Request") and
      this.asExpr() = ma
    )
  }
}
```

이제 `RemoteUserInput`(원격 사용자 입력)이라는 표준 집합에 우리의 `FastWebSource`가 포함되었습니다.
기존의 모든 SQL Injection, XSS 쿼리가 자동으로 이 새로운 소스를 인식하고 검사를 시작합니다! **마법 같죠.**

---

## 🛠️ Lab: The Missing Link

여러분이 선정한 타겟 중 하나를 골라보세요.
혹시 DB 쿼리를 날릴 때 `MyDatabase.exec(sql)` 같은 래퍼(Wrapper) 함수를 쓰나요?
CodeQL은 이게 Sink인지 모릅니다.

1.  `MyDatabase.exec`를 찾는 클래스를 작성하세요.
2.  `QueryExecution` (표준 Sink 클래스)을 상속받거나(`extends`), Taint Config의 `isSink`에 추가하세요.
3.  다시 스캔을 돌려보세요. 안 보이던 취약점이 우수수 떨어질 겁니다.

---

## 📝 7주차 과제: Source Modeling

본인이 사용하는 언어의 마이너한 웹 프레임워크나 라이브러리를 하나 찾아서,
그 라이브러리의 입력 함수(Source)를 정의하는 `.qll` (라이브러리 파일)을 작성하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** CodeQL이 기본적으로 인식하지 못하는 새로운 프레임워크나 라이브러리의 동작을 정의해주는 과정을 무엇이라 하나요? (M...)
2.  **Q2.** 사용자 정의 Source를 만들 때 상속받아야 하는 CodeQL 표준 클래스는 무엇인가요? (R... U... I...)
3.  **Q3.** 사용자 정의 쿼리를 모듈화하여 재사용하기 위해 저장하는 파일의 확장자는? (.q...)

---

다음 주는 **Analyzing Results at Scale**입니다.
파이프라인을 돌렸더니 결과(SARIF)가 5000개 나왔습니다. 이걸 언제 다 볼까요?
오탐(False Positive)을 걸러내고 알짜배기만 남기는 **데이터 분석**의 시간입니다.

**Teach the tool.**
