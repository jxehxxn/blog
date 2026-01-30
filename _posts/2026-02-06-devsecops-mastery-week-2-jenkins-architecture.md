---
layout: post
title: "DevSecOps Mastery: 2주차 - Jenkins 아키텍처와 보안의 기술"
---

지난 시간, 우리는 DevSecOps의 세계에 첫 발을 내디뎠습니다. Docker 위에 Jenkins를 띄우고, 첫 번째 "Hello World"를 외쳤던 그 설렘을 기억하시나요? 오늘은 그 장난감을 **'엔터프라이즈급 요새'**로 업그레이드할 시간입니다.

단순히 도구를 설치하는 것은 쉽습니다. 하지만 **"어떻게 안전하고 효율적으로 운영할 것인가?"**는 전혀 다른 문제입니다. 글로벌 빅테크 기업들이 Jenkins를 운영하는 철칙과 보안 원칙, 오늘 그 비밀을 파헤쳐 봅시다.

---

## 🏗️ 이론: Jenkins 아키텍처 해부

Jenkins는 혼자 일하지 않습니다. 거대한 공장을 지휘하는 지휘관과 실제 노동자로 나뉩니다.

### 1. Controller (Master): 두뇌 🧠
*   **역할:** 지휘관입니다. 일을 스케줄링하고, 파이프라인을 모니터링하고, 사용자 권한을 관리하며, 에이전트들에게 명령을 내립니다.
*   **특징:** 무거운 작업을 직접 하지 않습니다. (하지 말아야 합니다!)

### 2. Agents (Nodes): 근육 💪
*   **역할:** 실제 노동자입니다. 코드를 빌드하고, 테스트를 돌리고, 배포 스크립트를 실행하는 곳입니다.
*   **형태:** 물리 서버일 수도, 가상 머신일 수도, 우리가 지난주에 쓴 Docker 컨테이너일 수도 있습니다.

### ⚠️ 황금률: "Never Build on Master"
구글, 넷플릭스 같은 기업들의 제1원칙입니다. **"절대로 마스터(컨트롤러) 노드에서 빌드를 실행하지 마라."**

*   **이유 1 (보안):** 빌드 스크립트는 임의의 코드를 실행할 수 있습니다. 만약 악성 코드가 마스터에서 실행된다면? Jenkins 전체의 설정과 비밀키가 탈취될 수 있습니다.
*   **이유 2 (성능):** 마스터가 빌드하느라 CPU를 다 써버리면, 정작 중요한 스케줄링이나 웹 UI가 먹통이 됩니다.

> **우리의 실습 환경:** 현재는 Docker 하나에 마스터와 에이전트가 같이 있는 형태지만, 머릿속으로는 항상 이 둘을 분리해야 한다는 것을 명심하세요. 5주차에 Docker 에이전트를 분리하는 법을 배울 것입니다.

---

## 🧩 이론: 플러그인 생태계

Jenkins 본체는 깡통 로봇에 가깝습니다. 플러그인이 장착되어야 비로소 날아다니고 레이저를 쏠 수 있죠.

### Plugin Hell (플러그인 지옥) 🔥
"오, 이 기능 좋네?" 하고 무작정 플러그인을 수백 개 설치하면 어떻게 될까요?
*   Jenkins가 느려집니다.
*   플러그인끼리 충돌하여 업데이트할 때마다 에러가 터집니다.
*   보안 취약점이 기하급수적으로 늘어납니다.

**Pro Tip:** 꼭 필요한 플러그인만 설치하고, 정기적으로 정리하세요.

### 필수 추천 플러그인
*   **Blue Ocean:** 모던하고 예쁜 UI를 제공합니다.
*   **Git:** 소스 코드 관리를 위해 필수입니다.
*   **Credentials Binding:** 비밀번호, API 키를 안전하게 주입해 줍니다.
*   **Timestamper:** 로그에 시간을 찍어주어 디버깅을 돕습니다.

---

## 🔒 이론: 보안의 기술 (The Art of Security)

Jenkins는 회사의 소스 코드와 배포 권한을 가진 **심장부**입니다. 털리면 끝장입니다.

### 1. 인증(Authentication) vs 인가(Authorization)
*   **인증 (Who are you?):** 로그인 과정입니다. "나 홍길동이야."
*   **인가 (What can you do?):** 권한 부여입니다. "홍길동은 빌드는 할 수 있지만, 설정은 못 바꿔."

### 2. Credentials Management (비밀 관리)
**절대! NAVER!** 스크립트 안에 비밀번호를 하드코딩하지 마세요.
```groovy
// ❌ 최악의 예시
sh "docker login -u myid -p mypassword123"
```
소스 코드는 언젠가 유출될 수 있습니다. Jenkins의 **Credentials** 저장소에 암호화하여 저장하고, 파이프라인에서는 ID로만 호출해서 써야 합니다.

---

## 🛠️ 실습: 철통 보안 요새 만들기

자, 이제 우리의 Jenkins를 아무나 건드릴 수 없게 잠궈봅시다.

### Task 1: 역할 기반 권한 제어 (Role-based Strategy)
기본 Jenkins 권한 관리는 너무 단순합니다. 플러그인으로 고도화합시다.

1.  **Dashboard > Manage Jenkins > Plugins** 로 이동합니다.
2.  **Available plugins** 에서 `Role-based Authorization Strategy`를 검색하고 설치(Install without restart)합니다.
3.  **Manage Jenkins > Security** 로 이동합니다.
4.  **Authorization** 섹션에서 `Role-Based Strategy`를 선택하고 **Save**를 누릅니다.

### Task 2: Developer 역할 만들기
1.  **Manage Jenkins > Manage and Assign Roles** 메뉴가 새로 생겼습니다. 클릭!
2.  **Manage Roles** 클릭.
3.  **Global roles** 에 `developer`를 입력하고 Add.
4.  `developer` 행의 `Overall > Read`와 `Job > Build`, `Job > Read` 체크박스만 켭니다. (설정 권한 등은 주지 않습니다!)
5.  Save.

### Task 3: 주니어 개발자 계정 생성
1.  **Manage Jenkins > Users** > **Create User**.
2.  Username: `junior_dev`, Password: `password123` 등 입력 후 생성.
3.  다시 **Manage and Assign Roles** > **Assign Roles** 로 이동.
4.  **Global roles** 에서 `junior_dev` 사용자를 추가하고, 방금 만든 `developer` 역할에 체크합니다.
5.  Save.

이제 시크릿 탭을 열고 `junior_dev`로 로그인해보세요. **Manage Jenkins** 메뉴가 안 보일 겁니다. 이것이 바로 권한 분리입니다.

### Task 4: 안전한 비밀키 사용
1.  **Manage Jenkins > Credentials** > **System** > **Global credentials (unrestricted)**.
2.  **Add Credentials** 클릭.
3.  **Kind**: `Secret text` 선택.
4.  **Secret**: `super_secret_password_123` (여러분의 비밀값).
5.  **ID**: `my-api-key`.
6.  **Create**.

이제 이 비밀키는 Jenkins 내부에 암호화되어 저장되었습니다.

---

## 📝 2주차 과제: 익명 사용자 차단

여러분의 Jenkins에 로그아웃 상태로 접속해보세요. 혹시 작업 목록이 보이나요?
**Anonymous(익명) 사용자**가 아무것도 볼 수 없도록 권한을 설정하세요.

*   힌트: **Manage and Assign Roles** 설정에서 `Anonymous` 유저의 모든 체크박스를 해제하면 됩니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Jenkins에서 무거운 빌드 작업을 수행하는 노드를 무엇이라고 부르나요?
2.  **Q2.** "Never Build on Master" 원칙을 지켜야 하는 두 가지 주요 이유는 무엇인가요?
3.  **Q3.** 소스 코드에 비밀번호를 직접 적는 대신, Jenkins의 어떤 기능을 사용해야 하나요?

---

다음 주, 드디어 우리는 마우스 클릭질(ClickOps)에서 벗어나 **코드로 작성하는 파이프라인(Pipeline as Code)**의 세계로 진입합니다. 진정한 자동화가 시작됩니다.

**Stay Secure, Build Safely.**
