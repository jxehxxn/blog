---
layout: post
title: "DevSecOps Mastery: 9주차 - 지속적 배포(CD)와 IaC (Infrastructure as Code)"
---

개발(Dev)과 보안(Sec)을 챙겼으니, 이제 운영(Ops)의 꽃인 **배포(Deployment)**를 자동화할 차례입니다.
FTP로 파일을 올리거나, SSH로 접속해서 `git pull`을 치고 있나요?
오늘부로 그 습관은 역사 속으로 사라집니다.

---

## 🚚 CD: Delivery vs Deployment

용어가 비슷해서 헷갈리시죠?

1.  **Continuous Delivery (지속적 전달):** 언제든 배포할 수 있는 상태(아티팩트)를 만들지만, **최종 배포 버튼은 사람이** 누릅니다. (금융권, 대기업 등 안정성이 중요한 곳)
2.  **Continuous Deployment (지속적 배포):** 테스트를 통과하면 **자동으로** 고객에게 배포됩니다. (넷플릭스, 페이스북 등 속도가 중요한 곳)

우리는 Jenkins의 `input` 기능을 이용해 **Delivery** 모델(승인 후 배포)을 실습할 것입니다.

---

## 🏗️ Infrastructure as Code (IaC)

서버를 세팅할 때도 코드를 씁니다.
"AWS 콘솔 들어가서 EC2 만들고, 보안 그룹 열고..." -> **ClickOps (나쁨)**
"테라폼 파일에 `resource "aws_instance" ...` 적고 실행" -> **IaC (좋음)**

*   **Terraform:** 인프라 생성 (서버, 네트워크, DB)
*   **Ansible:** 설정 관리 (Nginx 설치, 설정 파일 복사)

---

## 🚦 실습: Automated Deployment (Mock)

실제 AWS를 쓰면 비용이 발생하므로, 쉘 스크립트로 배포 과정을 흉내(Mocking) 내보겠습니다. 하지만 원리는 똑같습니다.

### 1. 배포 스크립트 (`deploy.sh`)
```bash
#!/bin/bash
echo "🚀 Deploying to Production Server..."
echo "🔹 Stopping old service..."
sleep 2
echo "🔹 Copying new artifact..."
sleep 2
echo "🔹 Starting new service..."
sleep 2
echo "✅ Deployment Complete!"
```

### 2. Jenkinsfile에 CD 스테이지 추가

```groovy
stage('Deploy to Production') {
    // 오직 main 브랜치에서만 실행
    when { branch 'main' }
    
    steps {
        // 승인 절차 (Approval Gate)
        input message: 'Production 배포를 진행하시겠습니까?', ok: 'Deploy!'
        
        // 배포 스크립트 실행
        sh 'chmod +x deploy.sh'
        sh './deploy.sh'
    }
}
```

이제 파이프라인이 돌다가 이 단계에서 멈춥니다.
Jenkins UI에 "Paused for Input"이라고 뜨고, 여러분이 **[Deploy!]** 버튼을 눌러야만 배포가 진행됩니다. 이것이 바로 엔터프라이즈급 CD 파이프라인입니다.

---

## 📝 9주차 과제: 배포 전략 조사

배포 중에 서비스가 중단되면 안 되겠죠? 무중단 배포 전략 두 가지를 조사해서 정리해 보세요.
1.  **Blue/Green Deployment:** 파란색(구버전)과 초록색(신버전) 두 세트를 운영하는 방식.
2.  **Canary Deployment:** 광산의 카나리아처럼, 소수의 유저에게만 신버전을 먼저 노출하는 방식.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 지속적 전달(Delivery)과 지속적 배포(Deployment)의 결정적인 차이는 무엇인가요? (사람의 개입 여부)
2.  **Q2.** 서버 인프라 생성과 설정을 코드로 관리하는 기술을 약어로 무엇이라 하나요?
3.  **Q3.** Jenkins 파이프라인에서 사람의 승인을 기다리게 하는 명령어(Step)는 무엇인가요?

---

다음 주는 대망의 **Kubernetes (K8s)**입니다.
도커 컨테이너 하나는 쉽지만, 100개를 관리하려면 어떻게 해야 할까요?
오케스트레이션의 제왕, 쿠버네티스를 영접할 준비 하세요.

**Automate all the things.**
