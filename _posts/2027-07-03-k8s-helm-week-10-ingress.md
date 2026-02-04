---
layout: post
title: "Kubernetes Package Management with Helm: Week 10 - Ingress Controller & Traffic Management (Nginx)"
---

쿠버네티스 클러스터 안에 아무리 멋진 앱을 띄워놔도, 외부에서 접속을 못 하면 소용없습니다.
`NodePort`? `LoadBalancer`? IP가 부족하거나 비용이 비쌉니다.

정답은 **Ingress**입니다. 하나의 IP로 수천 개의 도메인을 처리하는 L7 라우터죠.
오늘은 사실상의 표준인 **Nginx Ingress Controller**를 Helm으로 배포하고 다루는 법을 배웁니다.

---

## 1. How Ingress Works

1.  **Ingress Resource (YAML):** "a.com은 서비스 A로, b.com은 서비스 B로 보내라"는 규칙.
2.  **Ingress Controller (Pod):** 위 규칙을 감시하다가, 변경되면 **Nginx 설정 파일(nginx.conf)을 다시 쓰고 재로딩(Reload)**합니다.
3.  **Load Balancer:** 클라우드 LB가 트래픽을 Ingress Controller로 던져줍니다.

---

## 2. Helm Integration: Nginx Ingress

`ingress-nginx` 차트는 설정할 게 정말 많습니다.

### 필수 설정 (values.yaml)
```yaml
controller:
  service:
    type: LoadBalancer # 클라우드 LB 사용
    annotations:
      # AWS 예시: NLB 사용
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
  metrics:
    enabled: true # Prometheus 연동
```

### 내 앱 차트에서의 Ingress
내 앱 차트(`templates/ingress.yaml`)에서는 사용자가 `values.yaml`에서 호스트네임을 바꿀 수 있게 해야 합니다.

```yaml
spec:
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          ...
    {{- end }}
```

---

## 3. Advanced Traffic Management

### Canary Release (카나리아 배포)
새 버전(v2)을 배포할 때, 전체 트래픽의 5%만 v2로 보내고 싶다면?
Nginx Ingress는 Annotation만으로 이걸 해줍니다.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "5"
```

### Rate Limiting (속도 제한)
DDoS 공격이나 과도한 크롤링을 막으려면?
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "5" # 초당 5회 제한
```

---

## 🛠️ Lab: Path-based Routing

하나의 도메인에서 경로(Path)에 따라 다른 서비스로 분기하는 실습입니다.

1.  **Ingress Controller 설치:** `helm install nginx ingress-nginx/ingress-nginx`
2.  **앱 2개 배포:** `apple` 앱과 `banana` 앱.
3.  **Ingress 작성:**
    *   `example.com/apple` -> `apple-service`
    *   `example.com/banana` -> `banana-service`
4.  **테스트:** `/etc/hosts`에 `example.com`을 Ingress IP로 등록하고 `curl`로 확인.

---

## 📝 10주차 과제: Blue/Green Deployment with Helm

**목표:** Helm 차트를 수정하여 Blue/Green 배포 구조를 만드세요.

1.  차트에 `deployment-blue.yaml`과 `deployment-green.yaml`을 만듭니다. (각각 다른 이미지 태그 사용 가능)
2.  `values.yaml`에 `activeColor: blue` 변수를 둡니다.
3.  `service.yaml`의 Label Selector가 `activeColor`를 바라보게 합니다.
    ```yaml
    selector:
      app: my-app
      color: {{ .Values.activeColor }}
    ```
4.  `helm upgrade --set activeColor=green`을 실행하면, 서비스가 즉시 Green 파드로 트래픽을 전환하는지 확인하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Ingress 리소스만 생성하고 Ingress Controller를 설치하지 않으면 어떻게 되나요? (아무 일도 일어나지 않음. 규칙을 수행할 주체가 없음)
2.  **Q2.** Nginx Ingress Controller에서 설정 변경 시 프로세스를 재시작하지 않고 설정을 적용하는 방식을 무엇이라 하나요? (Config Reload)
3.  **Q3.** 클라이언트의 실제 IP(Source IP)를 파드까지 전달하기 위해 `externalTrafficPolicy`를 무엇으로 설정해야 하나요? (`Local`)

---

다음 주, HTTP는 위험합니다. HTTPS(TLS)를 적용해야죠.
하지만 인증서 갱신은 귀찮습니다. **Cert-Manager**와 **Let's Encrypt**로 인증서를 평생 자동으로 갱신하는 마법을 배웁니다.

**Traffic Controlled.**
