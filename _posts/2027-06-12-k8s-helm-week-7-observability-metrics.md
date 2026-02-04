---
layout: post
title: "Kubernetes Package Management with Helm: Week 7 - Observability Stack I - Metrics (Prometheus + Grafana)"
---

쿠버네티스 운영의 핵심은 **관측 가능성(Observability)**입니다.
"서버가 느려요"라는 말을 듣고 Pod에 들어가서 `top`을 치고 있다면 이미 늦었습니다.

Part 2에서는 현대 모니터링의 표준 스택인 Prometheus와 Grafana를 내 차트에 통합하는 법을 배웁니다. 단순히 `kube-prometheus-stack`을 설치하는 게 아니라, **내 애플리케이션이 메트릭을 뱉게 하고, 그걸 수집(Scrape)하게 만드는 전체 파이프라인**을 구축합니다.

---

## 1. Prometheus Architecture

### Pull vs Push
전통적인 모니터링(Nagios, Zabbix)은 에이전트가 서버로 데이터를 보냅니다(Push).
Prometheus는 반대입니다. 서버가 주기적으로 앱에 접속해서 데이터를 긁어갑니다(**Pull**).

*   **Exporter:** 앱이 `/metrics` 경로로 현재 상태를 노출합니다.
*   **Prometheus Server:** 설정된 주기(예: 15초)마다 `/metrics`를 찔러서 데이터를 가져갑니다.

### Why Pull?
*   앱이 모니터링 서버 주소를 몰라도 됩니다. (Decoupling)
*   모니터링 서버가 죽어도 앱에는 영향이 없습니다.
*   수천 개의 마이크로서비스를 중앙에서 제어하기 쉽습니다.

---

## 2. Helm Integration: ServiceMonitor

Prometheus Operator를 쓴다면, `prometheus.yaml` 설정을 직접 건드릴 필요가 없습니다.
**ServiceMonitor**라는 CRD(Custom Resource Definition)만 만들면 됩니다.

"이 ServiceMonitor 리소스를 배포하면, Prometheus가 자동으로 내 앱을 감지해서 긁어가기 시작합니다."

```yaml
# templates/servicemonitor.yaml
{{- if .Values.metrics.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ .Release.Name }}
  labels:
    release: prometheus # Prometheus가 이 라벨을 보고 감지함 (중요!)
spec:
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  endpoints:
    - port: http
      path: /metrics
{{- end }}
```

---

## 3. Grafana Dashboard as Code

Grafana 대시보드를 마우스로 클릭해서 만들지 마세요.
JSON으로 Export해서 차트 안에 넣어야 합니다. 그래야 `helm install` 하자마자 대시보드가 짠 하고 나타납니다.

**ConfigMap으로 대시보드 배포하기:**
Grafana는 특정 라벨이 붙은 ConfigMap을 자동으로 읽어서 대시보드로 등록하는 사이드카 패턴을 지원합니다.

```yaml
# templates/dashboard-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-dashboard
  labels:
    grafana_dashboard: "1" # Grafana가 감지하는 라벨
data:
  my-dashboard.json: |
    {{ .Files.Get "dashboards/my-dashboard.json" }}
```

---

## 🛠️ Lab: Exposing Metrics

Node.js 앱에 메트릭을 붙여봅시다.

### 1. 앱 수정 (prom-client)
```javascript
const client = require('prom-client');
const collectDefaultMetrics = client.collectDefaultMetrics;
collectDefaultMetrics(); // CPU, Memory 등 기본 메트릭 수집

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

### 2. 차트 수정
*   `values.yaml`에 `metrics.enabled: true` 추가.
*   `templates/servicemonitor.yaml` 작성.

### 3. 확인
*   `helm install` 후.
*   Prometheus UI (Target 메뉴)에서 내 앱이 `UP` 상태인지 확인.
*   Grafana에서 `nodejs_heap_size_total_bytes` 쿼리로 차트가 그려지는지 확인.

---

## 📝 7주차 과제: Custom Metric

**목표:** 단순한 CPU/Mem 말고, 비즈니스 메트릭을 수집하세요.

1.  앱에 `http_request_duration_seconds`라는 **Histogram** 메트릭을 추가하세요. (API 응답 속도 측정)
2.  이 메트릭을 이용해 Grafana에서 **"API 99분위 응답 시간 (P99 Latency)"**을 보여주는 패널을 JSON으로 추출하세요.
3.  이 JSON을 Helm Chart에 포함시켜서 배포하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Prometheus가 애플리케이션에서 메트릭을 수집하는 방식을 무엇이라 하나요? (Pull 방식 / Scraping)
2.  **Q2.** Prometheus Operator 환경에서, "어떤 서비스를 스크랩할지" 정의하는 쿠버네티스 리소스 이름은? (ServiceMonitor)
3.  **Q3.** Grafana 대시보드를 영구적으로 저장하고 배포하기 위해 주로 사용하는 쿠버네티스 리소스는? (ConfigMap - 사이드카가 로드함)

---

다음 주, 메트릭만으로는 "왜" 느린지 알 수 없습니다.
에러 로그를 중앙에서 검색하고 분석하는 **PLG Stack (Promtail, Loki, Grafana)**과 **Fluentd**를 배웁니다.

**Measure everything.**
