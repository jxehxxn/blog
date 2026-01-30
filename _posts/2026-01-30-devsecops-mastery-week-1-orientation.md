---
layout: post
title: "DevSecOps Mastery: 1주차 - 오리엔테이션 및 기초 다지기"
---

반갑습니다, 여러분.

저는 지난 30년간 정보 보안의 최전선에서 수많은 공격을 막아내고, 또 무너지는 시스템들을 목격해온 여러분의 가이드입니다. 오늘부터 12주간, 우리는 단순한 개발자나 운영자를 넘어, **보안을 품은 엔지니어(DevSecOps Engineer)**로 거듭나는 여정을 시작합니다.

"보안은 전문가의 영역이다", "보안은 어렵다"라고 생각하시나요?  
틀렸습니다. 보안은 이제 **기본 소양**입니다.

이 강의는 컴퓨터 공학에 갓 입문한 초심자부터, 현업에서 한계를 느끼는 주니어 엔지니어까지 모두를 아우릅니다. 우리는 바닥부터 시작하여 견고한 성을 쌓아 올릴 것입니다. 12주 후, 여러분은 스스로 안전한 파이프라인을 구축하고 운영할 수 있는 **프로**가 되어 있을 것입니다.

자, 그 첫걸음을 내딛어 봅시다.

---

## 📅 12주 완성 DevSecOps 마스터리 커리큘럼

우리의 여정은 다음과 같이 계획되어 있습니다. 한 주도 빠짐없이 따라온다면, 여러분은 어느새 고지에 올라있을 겁니다.

| 주차 | 주제 | 주요 내용 |
|:---:|:---|:---|
| **1주차** | **오리엔테이션 & 기초 (Orientation & Foundations)** | DevSecOps 개념, 문화, Docker/Jenkins 환경 구축 |
| **2주차** | **Jenkins 아키텍처와 기본 설정 (Jenkins Architecture)** | Master-Agent 구조 이해, 플러그인 관리, 보안 설정 |
| **3주차** | **Pipeline as Code 기초 (Pipeline Basics)** | Jenkinsfile 작성법, Declarative Pipeline 문법 |
| **4주차** | **고급 파이프라인 문법 (Advanced Syntax)** | 조건문, 반복문, 병렬 실행, 함수 활용 |
| **5주차** | **Docker와 파이프라인의 만남 (Docker Integration)** | 도커 에이전트 활용, 컨테이너 빌드 및 배포 자동화 |
| **6주차** | **지속적 통합 실무 (CI Practices)** | 단위 테스트(Unit Test) 통합, 빌드 아티팩트 관리 |
| **7주차** | **DevSecOps 입문 & SAST (Static Analysis)** | 보안의 시작점, 소스 코드 정적 분석(SonarQube 등) 통합 |
| **8주차** | **컨테이너 보안 & DAST (Container Security)** | 이미지 스캔(Trivy), 동적 애플리케이션 보안 테스트 |
| **9주차** | **지속적 배포와 IaC (CD & Infrastructure as Code)** | Ansible/Terraform 맛보기, 자동 배포 파이프라인 |
| **10주차** | **Kubernetes 맛보기 (Kubernetes Basics)** | K8s 클러스터 연동, Helm 차트 배포 |
| **11주차** | **모니터링과 로깅 (Monitoring & Logging)** | ELK Stack, Prometheus/Grafana를 통한 가시성 확보 |
| **12주차** | **캡스톤 프로젝트 (Capstone Project)** | 처음부터 끝까지: 완전한 DevSecOps 파이프라인 구축 |

---

## 🎓 1주차 강의: 왜 DevSecOps인가?

### 1. 혼란의 벽 (The Wall of Confusion)

옛날 옛적, 소프트웨어 개발에는 거대한 벽이 하나 있었습니다.

*   **개발팀(Dev):** "나는 기능을 빨리 만들어야 해! 변화가 나의 목표야!" 🏃‍♂️
*   **운영팀(Ops):** "시스템은 안정적이어야 해! 변화는 장애의 원인이야!" 🛡️

개발팀이 코드를 다 짜서 벽 너머로 던져버리면, 운영팀은 그걸 받아서 힘겹게 서버에 올렸습니다. 에러가 나면 서로 삿대질을 했죠. "네 코드가 문제야!", "아니야, 네 서버 설정이 문제야!"

이것이 바로 **'혼란의 벽(The Wall of Confusion)'**입니다. **DevOps**는 이 벽을 허물기 위해 탄생했습니다. 개발과 운영이 하나의 팀처럼 소통하고 협력하는 문화, 그것이 DevOps의 핵심입니다.

### 2. 왼쪽으로 이동하라 (Shift Left)

DevOps로 벽을 허물었더니 속도가 엄청나게 빨라졌습니다. 그런데 문제가 생겼습니다. **보안(Security)**이 속도를 따라가지 못하는 겁니다.

전통적인 방식에서는 개발이 다 끝나고 배포 직전에 보안 검사를 했습니다.
> "자, 이제 출시해볼까?" -> (보안팀 등장) 👮‍♂️ "잠깐! 심각한 취약점이 100개 발견됐습니다. 출시 불가!"

결국 출시는 미뤄지고, 개발자는 몇 달 전 코드를 다시 수정해야 하는 악몽이 펼쳐집니다.

그래서 나온 개념이 **'Shift Left'**입니다. 타임라인의 오른쪽(배포 단계)에 있던 보안 절차를 왼쪽(개발/빌드 단계)으로 옮기자는 것입니다.

*   코드를 짜는 순간 검사하고,
*   빌드할 때 검사하고,
*   배포하기 전에 검사합니다.

보안이 파이프라인의 **일부**가 되는 것, 이것이 바로 **DevSecOps**입니다.

### 3. 자동차 공장 비유 (The Car Factory Analogy)

**CI/CD (Continuous Integration / Continuous Deployment)**를 자동차 공장의 컨베이어 벨트에 비유해 봅시다.

1.  **Code (부품 제작):** 개발자가 코드를 작성합니다. (자동차 문짝을 만듭니다.)
2.  **Commit (부품 전달):** 코드를 저장소에 올립니다. (컨베이어 벨트에 올립니다.)
3.  **CI (조립 및 품질 검사):** 로봇팔(Jenkins)이 자동으로 코드를 빌드하고 테스트합니다. (문짝이 맞는지 끼워보고, 도색이 잘 됐는지 검사합니다.)
4.  **DevSecOps (안전 검사):** 엑스레이(SAST)로 내부에 균열은 없는지, 충돌 테스트(DAST)로 안전한지 확인합니다.
5.  **CD (출고):** 모든 검사를 통과하면 자동으로 고객에게 배포됩니다. (완성차가 매장으로 이동합니다.)

우리는 이 '자동화된 공장'을 직접 설계하고 짓는 공장장이 될 것입니다.

---

## 🛠️ 실습: DevSecOps 연구실 구축하기

자, 이론은 충분합니다. 이제 손을 더럽힐 시간입니다.
우리의 모든 실습은 **Docker** 위에서 진행됩니다. 여러분의 컴퓨터를 더럽히지 않으면서도 완벽한 격리 환경을 제공하기 때문이죠.

### Step 1: Docker Desktop 설치

Docker는 컨테이너 기술의 핵심입니다. 가벼운 가상 컴퓨터라고 생각하면 쉽습니다.

1.  [Docker 공식 홈페이지](https://www.docker.com/products/docker-desktop/)에 접속합니다.
2.  여러분의 운영체제(Windows/Mac/Linux)에 맞는 버전을 다운로드하고 설치합니다.
3.  설치 완료 후 터미널(CMD, PowerShell, Terminal)을 열고 다음 명령어를 입력하여 확인합니다.

```bash
docker --version
# Docker version 24.0.x, build ... 와 같이 나오면 성공!
```

### Step 2: Jenkins 서버 띄우기 (Docker 활용)

우리는 Jenkins를 로컬에 직접 설치하지 않습니다. Docker를 이용해 단 한 줄의 명령어로 실행할 것입니다. 이것이 현대적인 방식입니다.

터미널에 다음 명령어를 복사해서 붙여넣으세요.

```bash
docker run -d -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home --name my-jenkins jenkins/jenkins:lts-jdk17
```

**명령어 해설:**
*   `docker run`: 컨테이너를 실행하라.
*   `-d`: 백그라운드에서 실행하라 (Detached mode).
*   `-p 8080:8080`: 내 컴퓨터의 8080 포트를 컨테이너의 8080 포트(Jenkins 웹)와 연결하라.
*   `-p 50000:50000`: 에이전트 연결을 위한 포트를 열어라.
*   `-v jenkins_home:/var/jenkins_home`: 젠킨스 데이터를 내 컴퓨터의 볼륨(jenkins_home)에 저장해라. (컨테이너가 삭제되어도 데이터는 남습니다.)
*   `--name my-jenkins`: 컨테이너 이름을 'my-jenkins'로 짓겠다.
*   `jenkins/jenkins:lts-jdk17`: Jenkins 공식 이미지 중 LTS(장기 지원) 버전, Java 17 포함 버전을 사용하겠다.

### Step 3: 초기 설정 마법사

1.  브라우저를 열고 `http://localhost:8080` 으로 접속합니다.
2.  **Unlock Jenkins** 화면이 나옵니다. 초기 비밀번호가 필요합니다.
3.  터미널로 돌아가서 다음 명령어로 비밀번호를 확인합니다.

```bash
docker exec my-jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

4.  출력된 복잡한 문자열을 복사해서 붙여넣습니다.
5.  **Install suggested plugins**를 선택합니다. (커피 한 잔 하세요. 시간이 좀 걸립니다 ☕)
6.  Admin 계정을 생성합니다. (ID/PW는 절대 잊어버리지 않게 메모하세요!)
7.  URL 설정은 그대로 두고 `Save and Finish` -> `Start using Jenkins`를 클릭합니다.

---

## 📝 1주차 과제: 첫 번째 Hello World

여러분의 첫 번째 Jenkins Job을 만들어 봅시다.

1.  Jenkins 메인 화면에서 **New Item**을 클릭합니다.
2.  이름에 `MyFirstJob`을 입력합니다.
3.  **Freestyle project**를 선택하고 OK를 누릅니다.
4.  아래로 스크롤하여 **Build Steps** 섹션을 찾습니다.
5.  **Add build step** -> **Execute shell**을 선택합니다.
6.  Command 창에 다음 스크립트를 입력합니다.

```bash
echo "Hello DevSecOps!"
echo "Today is $(date)"
echo "I am ready to learn."
```

7.  **Save**를 누릅니다.
8.  왼쪽 메뉴에서 **Build Now**를 클릭합니다.
9.  아래 **Build History**에 `#1`이 뜨고 파란불(성공)이 들어오면 클릭합니다.
10. **Console Output**을 눌러보세요. 우리가 입력한 텍스트가 출력되었나요?

축하합니다! 여러분은 방금 첫 CI 작업을 수행했습니다. 🎉

---

## ✅ Self-Assessment Quiz

오늘 배운 내용을 점검해 봅시다. (정답은 다음 시간에 공개합니다!)

1.  **Q1.** 개발팀과 운영팀 사이의 소통 단절과 갈등을 비유하는 용어는 무엇인가요?
2.  **Q2.** 보안 검사 과정을 배포 단계에서 개발 초기 단계로 앞당기는 접근 방식을 무엇이라 하나요?
3.  **Q3.** 우리가 설치한 Jenkins 컨테이너가 삭제되더라도 데이터를 보존하기 위해 Docker 명령어에서 사용한 옵션은 무엇인가요? (`-p`, `-d`, `-v` 중 선택)

---

다음 주에는 **Jenkins의 심장부(Architecture)**를 해부하고, 플러그인을 통해 기능을 확장하는 방법을 배웁니다.
준비되셨나요? 그럼 다음 주 강의실에서 뵙겠습니다.

**Keep Automating, Stay Secure.**
