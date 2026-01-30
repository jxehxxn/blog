---
layout: post
title: "DevSecOps Mastery: 8주차 - 컨테이너 보안과 DAST (동적 분석)"
---

지난주에 우리는 소스 코드를 깨끗하게 만들었습니다(SAST).
하지만 소스 코드가 깨끗하다고 해서, 운영 환경이 안전할까요?
여러분이 사용하는 `node:14` 이미지, `ubuntu:20.04` 이미지... 그 안에 해커가 좋아하는 구멍(CVE)이 숭숭 뚫려있다면 어떨까요?

오늘은 **컨테이너 자체의 보안**과, 실행 중인 앱을 공격해보는 **DAST**를 배웁니다.

---

## 📦 Container Security (이미지 스캔)

도커 이미지는 레이어(Layer)로 이루어져 있습니다. OS(Ubuntu/Alpine) 위에 라이브러리, 그 위에 앱이 올라가죠.
우리가 짠 코드는 맨 위 레이어일 뿐입니다. 아래쪽 레이어에서 취약점이 발견되면 다 뚫립니다.

### Tool: Trivy (by Aqua Security)
**Trivy**는 현존하는 가장 빠르고 정확한 컨테이너 스캐너입니다. 사용법도 허무할 정도로 간단합니다.

```bash
trivy image node:14
```
이 한 줄이면 OS 패키지 취약점부터 npm 라이브러리 취약점까지 싹 털어줍니다.

---

## 💥 DAST (Dynamic Application Security Testing)

**DAST**는 '블랙박스 테스팅'입니다.
내부를 모르는 상태에서, 실제 해커처럼 웹 사이트에 이상한 값도 넣어보고, 스크립트도 심어보면서 구멍을 찾습니다.

*   **언제 하나요?** 배포 직후(Staging 환경), 앱이 살아있을 때 합니다.
*   **Tool:** **OWASP ZAP (Zed Attack Proxy)**가 표준입니다.

---

## 🍔 실습: The Security Sandwich

보안 샌드위치를 만들어 봅시다. (빌드 전 스캔 + 배포 후 스캔)

### Layer 1: Container Scan (Pre-build)
Jenkinsfile에 Trivy 스캔을 추가합니다.

```groovy
stage('Container Scan') {
    steps {
        // 이미지를 파일시스템으로 스캔 (심각도 High 이상만 출력)
        sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity HIGH,CRITICAL my-app:v1'
        // --exit-code 1 옵션을 주면 취약점 발견 시 빌드를 실패시킬 수 있습니다.
    }
}
```

### Layer 2: DAST (Post-deploy)
앱을 잠깐 띄우고(Docker Run), ZAP으로 공격해 봅니다.

```groovy
stage('DAST Scan') {
    steps {
        // 1. 테스트용 앱 실행
        sh 'docker run -d -p 8080:3000 --name target-app my-app:v1'
        
        // 2. OWASP ZAP Baseline Scan 실행
        // (타겟 URL: http://host.docker.internal:8080)
        sh 'docker run --rm -t owasp/zap2docker-stable zap-baseline.py -t http://host.docker.internal:8080 || true'
        
        // 3. 앱 종료
        sh 'docker rm -f target-app'
    }
}
```
(주의: `|| true`를 붙인 이유는 ZAP이 경고만 찾아도 종료 코드를 1로 뱉어서 파이프라인이 죽는 것을 방지하기 위함입니다. 실무에서는 리포트를 보고 판단합니다.)

---

## 📝 8주차 과제: 버전별 취약점 비교

로컬 터미널에서 `Trivy`를 이용해 다음 두 이미지를 스캔해보고, **CRITICAL** 취약점 개수를 비교해 보세요.
1.  `node:10` (매우 구버전)
2.  `node:18-alpine` (최신 경량 버전)

결과를 보면 왜 우리가 베이스 이미지를 신중하게 골라야 하는지 뼈저리게 느끼게 될 겁니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 실행 중인 애플리케이션을 외부에서 해커처럼 공격하여 취약점을 찾는 테스트 방식을 무엇이라 하나요? (약어)
2.  **Q2.** 도커 이미지의 OS 패키지 및 라이브러리 취약점을 스캔하는 대표적인 오픈소스 도구는 무엇인가요? (T...)
3.  **Q3.** SAST와 DAST 중, 소스 코드가 없어도 수행할 수 있는 테스트는 무엇인가요?

---

다음 주는 **CD(지속적 배포)**와 **IaC(코드로서의 인프라)**입니다.
이제 만든 것을 자동으로 서버에 배포해야죠. 사람이 손으로 파일 복사하는 시대는 끝났습니다.

**Scan Early, Scan Often.**
