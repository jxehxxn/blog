---
layout: post
title: "Kubernetes Package Management with Helm: Week 1 - Orientation"
---

안녕하세요, 클라우드 네이티브 엔지니어 여러분.

지난 30년간 IT 인프라는 랙에 꽂힌 베어메탈 서버에서 가상 머신(VM)으로, 그리고 이제는 컨테이너와 쿠버네티스(Kubernetes)라는 거대한 오케스트레이션 시스템으로 진화했습니다.

하지만 쿠버네티스는 복잡합니다. YAML 파일 수백 줄을 관리하다 보면 "이걸 사람이 하라고 만든 건가?" 싶은 순간이 옵니다. 이때 등장한 구세주가 바로 **Helm**입니다. 쿠버네티스의 `apt-get`, `yum`, `pip` 같은 패키지 매니저죠.

오늘부터 12주간, 우리는 단순한 `helm install` 명령어를 넘어, **엔터프라이즈 수준의 Helm Chart를 직접 개발하고 배포하는 전문가**로 거듭날 것입니다. 여기에 더해 Skaffold, Prometheus, Grafana, Loki 등 모던 Observability 스택까지 통합하는 실전 기술을 다룹니다.

---

## 📅 12주 완성 커리큘럼 (Syllabus)

우리의 목표는 명확합니다. 12주 후, 여러분은 복잡한 마이크로서비스 아키텍처를 단 하나의 명령어로 배포하고 관리할 수 있게 될 것입니다.

### Part 1: Helm Fundamentals & Development (Helm 정복)
*   **W1:** 오리엔테이션 & Kubernetes/Helm 생태계 이해
*   **W2:** Helm 아키텍처와 기본 사용법 (Chart, Repository, Release)
*   **W3:** Chart 개발 기초 (Templates, Values.yaml, Chart.yaml)
*   **W4:** 고급 템플릿 로직 (Go Template, Flow Control, Named Templates)
*   **W5:** 의존성 관리와 서브차트 (Dependency Management, Subcharts)
*   **W6:** 로컬 개발 환경과 CI/CD (Skaffold, Helm Test)

### Part 2: Observability Stack Integration (관측 가능성 구축)
*   **W7:** Metrics: Prometheus & Grafana 차트 커스터마이징
*   **W8:** Logging: PLG Stack (Promtail, Loki, Grafana) + Fluentd
*   **W9:** Tracing: Jaeger & OpenTelemetry 통합

### Part 3: Traffic & Security (트래픽 및 보안)
*   **W10:** Ingress Controller (Nginx) & Traffic Management
*   **W11:** Security & Certificates (Cert-Manager, Let's Encrypt)
*   **W12:** Capstone: Production-Grade MSA 배포 파이프라인 구축

---

## 🎓 1주차 강의: Why Helm?

### 1. The "YAML Hell"
쿠버네티스를 처음 배우면 `kubectl apply -f deployment.yaml`이 마법처럼 느껴집니다. 하지만 서비스가 커지면 어떻게 될까요?
*   개발 환경(Dev)용 `deployment-dev.yaml`
*   운영 환경(Prod)용 `deployment-prod.yaml`
*   스테이징 환경(Staging)용 `deployment-staging.yaml`

내용은 99% 똑같은데, 이미지 태그랑 CPU 리소스 제한만 다릅니다. 이걸 복사-붙여넣기 하다 보면 실수가 발생합니다. "아, 운영 배포에 개발용 DB 주소를 넣었네..."

### 2. Templating Engine
Helm은 이 문제를 **템플릿(Template)**으로 해결합니다.
구멍 뚫린 틀(Template) 하나만 만들고, 환경별로 다른 값(Values)만 주입하는 방식입니다.

```yaml
# deployment.yaml (Template)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.appName }}
spec:
  replicas: {{ .Values.replicaCount }}
```

### 3. Package Management
더 중요한 건 **"패키지"**라는 개념입니다.
웹 서비스를 띄우려면 Deployment, Service, Ingress, ConfigMap, Secret 등이 한 세트로 필요합니다. Helm은 이 파일들을 `Chart`라는 하나의 꾸러미로 묶어서 관리합니다.
`helm install my-web-app ./my-chart` 명령어 한 방에 모든 리소스가 생성됩니다.

---

## 🛠️ Lab: Environment Setup

실습을 위해 로컬 쿠버네티스 환경을 구축합니다.

### 1. Tools Installation
다음 도구들을 설치해 오세요.
*   **Docker:** 컨테이너 런타임.
*   **Kind (Kubernetes in Docker) 또는 Minikube:** 로컬 쿠버네티스 클러스터. (M1/M2 맥북은 Docker Desktop의 Kubernetes도 좋습니다)
*   **Kubectl:** 쿠버네티스 CLI.
*   **Helm v3:** 오늘의 주인공.

### 2. Verify Installation
터미널에서 버전을 확인합니다.
```bash
docker version
kubectl version --client
helm version
```

### 3. First Helm Command
이미 만들어진 차트를 한번 설치해 봅시다. (Artifact Hub 이용)

```bash
# Bitnami 차트 저장소 추가
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Nginx 설치
helm install my-nginx bitnami/nginx

# 상태 확인
kubectl get pods
helm list
```

---

## 📝 1주차 과제: Kubernetes Resources 복습

Helm을 잘 쓰려면 결국 Kubernetes 리소스(YAML)를 잘 알아야 합니다.
다음 리소스들이 각각 무슨 역할을 하는지 1~2줄로 요약하여 제출하세요.

1.  **Pod**
2.  **Deployment**
3.  **Service (ClusterIP vs NodePort vs LoadBalancer)**
4.  **Ingress**
5.  **ConfigMap & Secret**
6.  **StatefulSet** (Deployment와의 차이점 위주로)

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Helm v2와 v3의 가장 큰 아키텍처 차이점은 무엇인가요? (힌트: Tiller)
2.  **Q2.** Helm Chart에서 변수들이 저장되는 기본 파일의 이름은 무엇인가요? (`values.yaml`)
3.  **Q3.** `helm install`과 `helm upgrade`의 차이는 무엇인가요?

---

다음 주, 우리는 Helm의 내부 구조를 뜯어보고, 직접 첫 번째 Chart를 만들어볼 것입니다.
YAML 복사-붙여넣기의 지옥에서 탈출할 준비 되셨나요?

**Happy Helming.**
