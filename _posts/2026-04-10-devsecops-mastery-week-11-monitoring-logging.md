---
layout: post
title: "DevSecOps Mastery: 11주차 - 모니터링과 로깅, 시스템의 눈"
---

개발(Dev)은 "Day 1"입니다. 시스템을 만들어서 배포하는 날이죠.
운영(Ops)은 **"Day 2"**입니다. 배포한 다음 날부터 영원히 계속되는 날들이죠.

DevOps의 철학 중 하나는 **"You build it, you run it"**입니다.
내가 만든 서비스가 아픈지 안 아픈지 알기 위해서는 청진기와 CCTV가 필요합니다.
그것이 바로 **모니터링**과 **로깅**입니다.

---

## 🔭 The Observability Triad (관측 가능성 3대장)

시스템을 완벽하게 파악하려면 세 가지 데이터가 필요합니다.

1.  **Logs (로그):** "무슨 일이 일어났어?" (What)
    *   예: "Error: NullPointerException at line 42"
    *   도구: **ELK Stack** (Elasticsearch, Logstash, Kibana), Loki
2.  **Metrics (메트릭):** "상태가 어때?" (Status)
    *   예: "CPU 사용량 90%", "초당 요청 수(RPS) 500개"
    *   도구: **Prometheus**, **Grafana**
3.  **Traces (트레이스):** "어디가 느려?" (Where)
    *   예: "사용자 클릭 -> A서버(10ms) -> B서버(2000ms) -> DB(5ms)" (B서버가 범인!)
    *   도구: Jaeger, Zipkin

---

## 📊 Lab: The Grafana Dashboard (Jenkins 모니터링)

우리의 CI 도구인 Jenkins도 모니터링 대상입니다.
Jenkins가 느려지면 개발팀 전체가 멈추니까요.

### 1. Prometheus Metrics Plugin 설치
Jenkins 관리 > 플러그인 관리에서 `Prometheus metrics plugin`을 설치합니다.
그러면 `http://jenkins-url/prometheus/` 경로로 메트릭 데이터가 쏟아져 나옵니다.

### 2. Grafana 연동 (개념적 설명)
Grafana를 설치하고 데이터 소스로 Jenkins를 등록하면 멋진 대시보드를 꾸밀 수 있습니다.
*   **Build Duration:** 빌드 시간이 점점 늘어나고 있나?
*   **Failure Rate:** 최근 빌드 실패율이 치솟았나?
*   **Queue Length:** 빌드를 기다리는 대기열이 너무 긴가?

이런 지표를 보고 "아, 에이전트를 더 늘려야겠구나"라고 판단하는 것이 **데이터 기반 엔지니어링**입니다.

---

## 🚨 Alert Fatigue (알람 피로)

"늑대와 양치기 소년" 이야기를 아시나요?
너무 잦은 알람은 독입니다.
*   CPU 80% 넘었다고 새벽 3시에 전화가 온다면?
*   알고 보니 잠깐 튀었다가 내려간 거라면?

결국 엔지니어는 알람을 끄게 되고, 진짜 장애가 났을 때 놓치게 됩니다.
**Actionable Alert (조치 가능한 알람)**만 설정하는 것이 핵심입니다.

---

## 📝 11주차 과제: 알람 피로 예방

여러분이 운영자라고 생각하고, 다음 상황에서 알람을 보낼지 말지 결정해 보세요.
1.  디스크가 90% 찼을 때 (O/X)
2.  디스크가 99% 찼을 때 (O/X)
3.  개발 서버가 다운되었을 때 (O/X)

(정답은 없지만, 보통 1번은 경고 메일, 2번은 새벽 전화, 3번은 무시(출근해서 해결)입니다.)

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 시스템의 상태를 나타내는 숫자 데이터(CPU, 메모리 등)를 무엇이라 하나요?
2.  **Q2.** 로그 데이터를 수집하고 검색하고 시각화하는 가장 유명한 오픈소스 스택의 이름은? (ELK...)
3.  **Q3.** 너무 많은 알람 때문에 운영자가 알람에 무감각해지는 현상을 무엇이라 하나요?

---

다음 주는 대장정의 마지막, **Capstone Project**입니다.
지금까지 배운 모든 것(Jenkins, Docker, SAST/DAST, K8s, Notification)을 하나로 합쳐서
완벽한 **End-to-End DevSecOps Pipeline**을 구축합니다.

**Observe, Don't Guess.**
