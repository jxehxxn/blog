---
layout: post
title: "Kubernetes Package Management with Helm: Week 9 - Observability Stack III - Tracing (Jaeger)"
---

마이크로서비스 아키텍처(MSA)의 악몽: **"범인 찾기"**.
User -> Front -> Backend -> Auth -> DB...
이 긴 여정 중 어디서 느려졌는지, 어디서 에러가 났는지 로그만 보고 찾는 건 사막에서 바늘 찾기입니다.

**Distributed Tracing(분산 추적)**은 요청 하나에 ID(Trace ID)를 붙여서 전체 경로를 추적하는 기술입니다.
오늘은 **Jaeger**와 **OpenTelemetry**를 배웁니다.

---

## 1. Trace, Span, Context Propagation

용어부터 정리합시다.

*   **Trace:** 요청 하나가 시스템 전체를 여행하는 전체 경로. (하나의 Trace ID를 공유)
*   **Span:** 경로의 각 구간. (예: DB 쿼리 실행 구간, HTTP 요청 구간)
*   **Context Propagation:** 서비스 A가 서비스 B를 호출할 때, "야, 이거 Trace ID 123번 작업이야"라고 헤더(`uber-trace-id`)에 담아 알려주는 것. 이게 끊기면 추적도 끊깁니다.

---

## 2. Jaeger Architecture

*   **Agent:** (Deprecated) 각 노드에 떠서 Spans을 수집해서 Collector로 보냄. 요즘은 OpenTelemetry Collector가 대신함.
*   **Collector:** Spans을 받아서 DB(Elasticsearch/Cassandra)에 저장.
*   **Query (UI):** 저장된 Trace를 시각화해서 보여줌.

---

## 3. Helm Integration: Sidecar vs Agent

내 앱에 Tracing을 적용하려면 두 가지가 필요합니다.
1.  **Code Instrumentation:** 코드에 추적 코드를 심어야 함. (OpenTelemetry SDK 사용)
2.  **Collector 배포:** Helm으로 Jaeger/OTEL Collector 설치.

### Auto-Instrumentation (Zero Code)
Java나 Python, Node.js는 코드 수정 없이도 에이전트만 붙이면 자동으로 추적해줍니다.
쿠버네티스에서는 **OpenTelemetry Operator**가 파드에 자동으로 사이드카를 주입(Injection)해줍니다.

```yaml
# Annotations만 붙이면 됨!
annotations:
  instrumentation.opentelemetry.io/inject-java: "true"
```

---

## 🛠️ Lab: HotROD (Ride on Demand)

Jaeger 팀에서 만든 데모 앱인 HotROD를 배포해서 트레이싱을 체험해봅니다.

1.  **Jaeger 설치:** `helm install jaeger jaegertracing/jaeger`
2.  **HotROD 배포:**
    ```bash
    kubectl run hotrod --image=jaegertracing/example-hotrod:latest --port=8080
    kubectl expose pod hotrod --type=NodePort
    ```
3.  **체험:**
    *   웹 브라우저로 HotROD 앱에 접속해서 버튼을 몇 번 누릅니다.
    *   Jaeger UI에 접속해서 `frontend` 서비스를 검색합니다.
    *   타임라인 차트(Gantt Chart)를 보면서 "Redis 호출이 500ms 걸렸네?" 등을 확인합니다.

---

## 📝 9주차 과제: OpenTelemetry Instrumentation

**목표:** Week 7~8에서 사용한 Node.js 앱에 OpenTelemetry를 적용하여 Jaeger로 Trace를 보내세요.

1.  `@opentelemetry/sdk-node` 등의 패키지를 설치합니다.
2.  앱 시작 시 `NodeSDK`를 초기화하고 `JaegerExporter`를 설정합니다.
3.  `helm install` 시 환경 변수로 Jaeger Agent 주소(`OTEL_EXPORTER_JAEGER_ENDPOINT`)를 주입하세요.
4.  Jaeger UI에서 내 앱의 Trace가 보이는 스크린샷을 제출하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 서비스 A가 서비스 B를 호출할 때 Trace ID를 전달하기 위해 HTTP 헤더를 주입하는 과정을 무엇이라 하나요? (Context Propagation)
2.  **Q2.** Jaeger 구성 요소 중, 수집된 데이터를 영구 저장소(DB)에 쓰는 역할을 하는 컴포넌트는? (Collector)
3.  **Q3.** 코드 수정 없이 라이브러리 레벨에서 자동으로 Span을 생성해주는 기능을 무엇이라 하나요? (Auto-Instrumentation)

---

이것으로 Part 2: 관측 가능성 스택이 완성되었습니다.
이제 우리 시스템은 눈과 귀를 가졌습니다.
다음 주부터는 외부 세계와의 관문, **Ingress**와 **Traffic Management**를 배웁니다.

**Follow the trace.**
