---
layout: post
title: "DevSecOps Mastery: 10주차 - Kubernetes (K8s), 오케스트레이션의 시대"
---

도커(Docker)가 컨테이너라는 '화물 상자'를 발명했다면,
**쿠버네티스(Kubernetes, K8s)**는 그 상자들을 싣고 나르는 '거대한 자동화 선박'입니다.

현대적인 클라우드 네이티브 환경에서 K8s를 모르면 대화가 통하지 않습니다.
오늘은 K8s의 핵심 개념을 잡고, Jenkins로 K8s에 배포하는 방법을 알아봅니다.

---

## ☸️ Why K8s?

컨테이너 하나를 띄우는 건 `docker run`이면 충분합니다.
하지만 컨테이너가 100개, 1000개가 된다면?
*   어떤 서버에 배치할지? (Scheduling)
*   죽으면 누가 살릴지? (Self-healing)
*   트래픽이 몰리면 어떻게 늘릴지? (Auto-scaling)

이걸 다 해주는 게 쿠버네티스입니다.

---

## 🏗️ Core Concepts (핵심 3대장)

1.  **Pod (파드):** K8s의 가장 작은 단위입니다. 컨테이너 하나(또는 그 이상)를 감싸고 있습니다. "프로세스"라고 생각하면 됩니다.
2.  **Deployment (디플로이먼트):** 파드의 복제본(Replica)을 관리합니다. "파드 3개를 항상 유지해!"라고 명령하면, 하나가 죽어도 알아서 다시 살려냅니다.
3.  **Service (서비스):** 파드들은 IP가 자꾸 바뀝니다. 고정된 IP나 도메인으로 파드들에 접근할 수 있게 해주는 "전화번호부"이자 "로드밸런서"입니다.

---

## 🚀 CD to K8s (Jenkins와 K8s의 만남)

Jenkins가 빌드한 도커 이미지를 K8s 클러스터에 배포하려면 `kubectl`이라는 명령어를 씁니다.
그리고 배포 명세서인 YAML 파일이 필요합니다.

### deployment.yaml 예시
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3 # 3개를 띄워라
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:v1 # Jenkins가 만든 이미지
        ports:
        - containerPort: 3000
```

---

## 🛠️ 실습: Deploying to Minikube (Dry-run)

실제 클러스터가 없어도 괜찮습니다. 문법 검사(Dry-run)로 실습해 봅시다.

1.  프로젝트에 위 `deployment.yaml` 파일을 만드세요.
2.  Jenkinsfile에 배포 스테이지를 추가합니다.

```groovy
stage('Deploy to K8s') {
    steps {
        // --dry-run=client: 실제로 적용하진 말고 검사만 해봐
        sh 'kubectl apply --dry-run=client -f deployment.yaml'
        echo 'K8s Deployment manifest is valid!'
    }
}
```

(실제 환경에서는 Jenkins 에이전트에 `kubectl`이 설치되어 있어야 하고, K8s 클러스터 접속 정보(`kubeconfig`)가 세팅되어 있어야 합니다.)

---

## 📝 10주차 과제: Service 타입 조사

K8s Service에는 여러 타입이 있습니다. 다음 두 가지의 차이점을 조사하세요.
1.  **ClusterIP:** (기본값) 클러스터 내부에서만 접근 가능.
2.  **LoadBalancer:** 외부(인터넷)에서 접근 가능하도록 클라우드 로드밸런서를 붙임.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 쿠버네티스에서 컨테이너를 실행하는 가장 작은 배포 단위는 무엇인가요?
2.  **Q2.** 파드의 개수를 유지하고, 배포 버전을 관리해주는 상위 컨트롤러의 이름은 무엇인가요?
3.  **Q3.** Jenkins에서 쿠버네티스 클러스터에 명령을 내리기 위해 사용하는 CLI 도구의 이름은?

---

다음 주는 **모니터링(Monitoring)과 로깅(Logging)**입니다.
시스템이 잘 돌아가는지 감시하지 않으면, 장애가 나도 고객이 전화할 때까지 모릅니다.
K8s 환경에서의 가시성(Observability) 확보 전략을 배웁니다.

**Orchestrate or Die.**
