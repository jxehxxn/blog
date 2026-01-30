---
layout: post
title: "DevSecOps Mastery: 12주차 - 캡스톤 프로젝트, 그리고 새로운 시작"
---

축하합니다! 🎉
장장 12주간의 대장정을 무사히 마치신 여러분께 뜨거운 박수를 보냅니다.
Docker 설치부터 시작해서 Kubernetes 배포와 모니터링까지, 여러분은 이제 어디 가서 "나 DevSecOps 좀 알아"라고 말할 자격이 충분합니다.

하지만 진정한 마스터가 되기 위한 마지막 관문이 남았습니다.
바로 **Capstone Project**입니다.

---

## 🎓 The Capstone Project (Final Exam)

여러분의 임무는 **"완전한 DevSecOps 파이프라인 구축"**입니다.
지금까지 배운 모든 조각을 하나로 맞추세요.

### 📋 요구사항 (Requirements)
다음 단계가 모두 포함된 `Jenkinsfile`을 작성하여 제출하세요.

1.  **SCM:** Git에서 코드를 가져옵니다.
2.  **Build:** Dockerfile을 이용해 이미지를 빌드합니다.
3.  **Test:** 유닛 테스트를 수행하고 JUnit 리포트를 발행합니다.
4.  **SAST:** 소스 코드 보안 검사를 수행합니다. (Semgrep 등) -> **Critical 발견 시 빌드 중단.**
5.  **Container Security:** 이미지 취약점 스캔을 수행합니다. (Trivy 등) -> **High 발견 시 빌드 중단.**
6.  **Push:** 안전한 이미지를 레지스트리(Docker Hub 등)에 푸시합니다.
7.  **CD (Staging):** 스테이징 환경(또는 Mock)에 자동 배포합니다.
8.  **CD (Production):** **승인(Approval)** 절차를 거친 후 프로덕션에 배포합니다.
9.  **Notification:** 빌드 성공/실패 여부를 슬랙이나 이메일, 또는 `echo`로 알립니다.

이 파이프라인이 성공적으로 돌아가는 로그(Log)가 여러분의 졸업장입니다.

---

## 🛣️ The Road Ahead (앞으로의 길)

이 강의는 끝났지만, 여러분의 여정은 이제 시작입니다.
DevSecOps의 세계는 넓고도 깊습니다. 다음 단계로 무엇을 공부하면 좋을까요?

1.  **Public Cloud:** AWS, Azure, Google Cloud의 관리형 DevOps 도구들을 익히세요.
2.  **Advanced K8s:** 서비스 메시(Istio), 오퍼레이터 패턴 등을 파보세요.
3.  **GitOps:** Jenkins가 아닌, ArgoCD를 이용해 Git 상태를 클러스터에 동기화하는 최신 트렌드를 익히세요.

---

## 👋 Closing

보안은 귀찮은 것이 아닙니다.
보안은 개발을 느리게 하는 방지턱이 아닙니다.
제대로 된 DevSecOps 파이프라인은, 개발자가 **"마음 놓고 속도를 낼 수 있게 해주는 가드레일"**입니다.

여러분이 구축한 파이프라인 덕분에, 동료들은 두려움 없이 코드를 배포하고, 회사는 해킹의 위협에서 안전해질 것입니다.
그 자부심을 가지고 현업으로 나아가세요.

지난 12주간 정말 고생 많으셨습니다.
여러분의 앞날에 무한한 `Build Success`가 함께 하기를!

**Keep Learning, Keep Automating.**

---
*Professor. Antigravity*
