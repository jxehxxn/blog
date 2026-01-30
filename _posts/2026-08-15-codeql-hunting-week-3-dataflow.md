---
layout: post
title: "Vulnerability Hunting with CodeQL: Week 3 - Following the Breadcrumbs (Data Flow & Taint Tracking)"
---

보안 취약점의 90%는 한 문장으로 요약됩니다.
**"신뢰할 수 없는 데이터(Untrusted Data)가 위험한 곳(Sink)으로 들어간다."**

*   SQL Injection: 사용자 입력 -> SQL 쿼리 실행
*   XSS: 사용자 입력 -> HTML 렌더링
*   RCE: 사용자 입력 -> 쉘 명령어 실행

이 흐름을 추적하는 기술을 **Taint Tracking(오염 추적)**이라고 합니다. CodeQL이 가장 잘하는 분야입니다.

---

## 🌊 Concepts: Source, Sink, Sanitizer

1.  **Source (수원지):** 데이터가 들어오는 곳. (예: `request.getParameter()`, `args[]`)
    *   여기서 들어온 데이터는 "Tainted(오염됨)" 상태입니다.
2.  **Sink (하수구):** 데이터가 사용되는 위험한 곳. (예: `executeQuery()`, `eval()`)
3.  **Sanitizer (정화조):** 오염을 제거하는 곳. (예: `Integer.parseInt()`, `escapeHtml()`)
    *   이곳을 통과하면 데이터는 다시 "Clean" 상태가 됩니다.

---

## 🛤️ Configuration (설정)

CodeQL에서는 이 흐름을 `TaintTracking::Configuration` (또는 최신 `DataFlow::Global`)으로 정의합니다.

```ql
import java
import semmle.code.java.dataflow.TaintTracking

class SqlInjectionConfig extends TaintTracking::Configuration {
  SqlInjectionConfig() { this = "SqlInjectionConfig" }

  override predicate isSource(DataFlow::Node source) {
    // 소스 정의: 원격 사용자 입력
    source instanceof RemoteUserInput
  }

  override predicate isSink(DataFlow::Node sink) {
    // 싱크 정의: SQL 실행 메소드의 첫 번째 인자
    exists(MethodCall ma |
      ma.getMethod().hasName("executeQuery") and
      sink.asExpr() = ma.getArgument(0)
    )
  }
}

from SqlInjectionConfig cfg, DataFlow::Node source, DataFlow::Node sink
where cfg.hasFlow(source, sink)
select sink, source, sink, "SQL Injection found!"
```
이 쿼리 하나면, 수만 줄의 코드 중에서 사용자 입력이 DB 쿼리로 직행하는 경로만 딱 찾아줍니다.

---

## 🛠️ Lab: Catching SQL Injection

준비된 취약한 샘플 앱(Java/Spring Boot 권장)에 위 쿼리를 돌려보세요.
VSCode의 "CodeQL: Run Query"를 실행하면, 결과창에서 **Source부터 Sink까지의 경로(Path)**를 시각적으로 보여줍니다.

1.  `controller.java:30` - `username` 파라미터 받음 (Source)
2.  `service.java:50` - `generateSql(username)` 호출
3.  `dao.java:20` - `stmt.executeQuery(sql)` 실행 (Sink)

이 경로가 보인다면 성공입니다.

---

## 📝 3주차 과제: Log Injection 찾기

SQL Injection 설정(Config)을 복사해서 수정해보세요.
*   **Source:** 그대로 (`RemoteUserInput`)
*   **Sink:** 로그를 남기는 함수 (예: `logger.info()`, `System.out.println()`)

사용자 입력이 로그 파일에 그대로 기록되면 **Log Injection** (또는 Log Forging) 취약점입니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 외부에서 들어온 검증되지 않은 데이터를 뜻하는 용어는? (S...)
2.  **Q2.** 오염된 데이터가 실행되어 실제 피해를 입히는 지점을 뜻하는 용어는? (S...)
3.  **Q3.** 오염된 데이터를 안전한 상태로 정화해주는 함수나 로직을 무엇이라 하나요?

---

다음 주는 **Control Flow(제어 흐름)**와 **AST(구문 트리)**입니다.
"데이터가 흘러가긴 하는데, 중간에 `if (isAdmin)` 체크를 했으면 안전한 거 아냐?"
이런 문맥(Context)을 파악하는 방법을 배웁니다.

**Follow the flow.**
