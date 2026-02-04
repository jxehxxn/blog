---
layout: post
title: "Kubernetes Package Management with Helm: Week 8 - Observability Stack II - Logging (Loki + Fluentd)"
---

"파드가 죽었어요." "로그 확인해봤어?" "아뇨, 파드가 사라져서 로그도 같이 날아갔는데요."

컨테이너 환경에서 로그는 파일에 남기면 안 됩니다. 표준 출력(Stdout)으로 뱉고, 누군가가 낚아채서 안전한 저장소로 보내야 합니다.
오늘은 로그계의 샛별 **Loki**와 전통의 강자 **Fluentd**를 배웁니다.

---

## 1. The Logging Architecture

### Node-Level Logging Agent
모든 파드에 사이드카를 붙이는 건 비효율적입니다.
대신 각 노드(Node)마다 **DaemonSet**으로 로그 수집기를 하나씩 띄웁니다.
이 수집기는 노드의 `/var/log/containers/*.log` 파일을 꼬리물기(tail)하여 모든 컨테이너의 로그를 읽어갑니다.

### Fluentd vs Promtail
*   **Fluentd:** 강력한 파싱/필터링 기능. 다양한 Output 지원 (Elasticsearch, S3, Kafka). 무거움(Ruby).
*   **Promtail:** Loki 전용 수집기. 가벼움(Go). 라벨 기반의 검색 지원.

우리는 **PLG Stack (Promtail + Loki + Grafana)**을 중심으로 배웁니다. Elasticsearch보다 훨씬 싸고 가볍거든요.

---

## 2. Loki: Like Prometheus, but for Logs

Loki의 철학은 독특합니다.
"로그 내용(Content)은 인덱싱하지 말자. 메타데이터(Label)만 인덱싱하자."

*   **Elasticsearch:** 모든 단어를 역인덱싱함. 빠르지만 스토리지 비용 폭발.
*   **Loki:** `app=nginx`, `env=prod` 같은 라벨만 인덱싱. 로그 본문은 그냥 압축해서 저장(Chunk). 검색할 땐 `grep` 처럼 훑음(Brute force).

최근 분산 환경에서는 이게 훨씬 효율적입니다.

---

## 3. Helm Integration: Log Parsing

로그는 그냥 텍스트 덩어리가 아닙니다. 구조화(JSON)되어야 분석할 수 있습니다.
앱이 JSON으로 로그를 뱉게 하고, Promtail이 이를 파싱하게 설정해야 합니다.

```yaml
# values.yaml (Promtail Chart)
config:
  snippets:
    pipelineStages:
      - docker: {}
      - json:
          expressions:
            level: level
            msg: message
      - labels:
          level: # 로그 레벨을 라벨로 승격
```

이렇게 하면 Grafana에서 `{app="my-app"} | json | level="error"` 처럼 쿼리할 수 있습니다.

---

## 🛠️ Lab: Deploying PLG Stack

1.  **Loki 설치:** `helm install loki grafana/loki-stack`
2.  **앱 배포:** Week 7에서 만든 앱을 배포합니다. 일부러 에러를 발생시키는 API(`/error`)를 만드세요.
3.  **Grafana 접속:**
    *   Data Source에 Loki 추가 (`http://loki:3100`).
    *   Explore 탭에서 `{app="my-app"}` 쿼리 실행.
    *   실시간으로 로그가 들어오는 것을 확인.

---

## 📝 8주차 과제: Fluentd to S3

Loki는 단기 분석용입니다. 규제 준수(Compliance)를 위해 로그를 1년간 보관해야 한다면? S3로 보내야 합니다.

**목표:** Fluentd Bitnami 차트를 사용하여, 모든 파드의 로그를 AWS S3 버킷(혹은 MinIO)으로 전송하는 설정을 `values.yaml`로 구성하세요.

1.  Input: `tail` (/var/log/containers/*.log)
2.  Filter: `kubernetes_metadata` (파드 이름, 네임스페이스 정보 추가)
3.  Output: `s3` 플러그인 설정.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Loki가 Elasticsearch에 비해 스토리지 비용이 저렴한 이유는? (전체 텍스트 인덱싱을 하지 않고 라벨만 인덱싱하기 때문)
2.  **Q2.** 쿠버네티스 노드마다 하나씩 실행되어 해당 노드의 모든 컨테이너 로그를 수집하는 파드 배포 방식을 무엇이라 하나요? (DaemonSet)
3.  **Q3.** Promtail 파이프라인 스테이지 중, 로그 내용에서 특정 값을 추출하여 검색 가능한 인덱스(Label)로 만드는 스테이지는? (`labels` stage)

---

다음 주, 로그와 메트릭으로도 잡을 수 없는 병목 구간을 찾습니다.
"A 서비스에서 B 서비스 호출하는 데 3초 걸렸는데, 정확히 어디서 느린 거야?"
이 질문에 답하는 **Distributed Tracing (Jaeger)**를 배웁니다.

**Log responsibly.**
