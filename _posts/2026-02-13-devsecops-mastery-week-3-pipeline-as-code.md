---
layout: post
title: "DevSecOps Mastery: 3주차 - Pipeline as Code, 자동화의 심장"
---

안녕하세요, DevSecOps 엔지니어 여러분.

지금까지 우리는 Jenkins 웹 화면에서 버튼을 꾹꾹 눌러가며(ClickOps) 작업을 만들었습니다. 편하긴 하지만, 만약 Jenkins 서버가 날아간다면? 그 많은 설정을 기억해서 다시 세팅할 수 있을까요? 혹은 동료가 "그 설정 어떻게 했어?"라고 물어본다면 캡처를 떠서 보여줘야 할까요?

오늘은 이 비효율을 끝내는 날입니다. 우리는 이제 **인프라를 코드처럼(Infrastructure as Code)** 다루듯, **파이프라인도 코드(Pipeline as Code)**로 다룰 것입니다.

---

## 🛑 The Problem with "ClickOps"

GUI(그래픽 유저 인터페이스)로 설정을 관리하면 생기는 치명적인 문제점들이 있습니다.

1.  **히스토리가 없다:** 누가 언제 어떤 설정을 바꿨는지 추적할 수 없습니다. (어제 잘 되던 빌드가 오늘 안 되는데, 누가 뭘 건드린 거지?)
2.  **재사용 불가:** 똑같은 파이프라인을 다른 프로젝트에 적용하려면 또 마우스질을 반복해야 합니다.
3.  **복구 불가:** 서버가 터지면 설정도 함께 증발합니다.

## 💡 The Solution: Jenkinsfile

이 모든 문제의 해결책은 **`Jenkinsfile`**이라는 텍스트 파일 하나입니다.
우리는 빌드, 테스트, 배포의 모든 절차를 이 파일에 적어서, 애플리케이션 소스 코드(Git)와 함께 저장할 것입니다.

*   코드로 되어 있으니 **버전 관리(Git)**가 됩니다.
*   누가 수정했는지 **추적(Blame)**이 가능합니다.
*   다른 프로젝트에 **복사-붙여넣기**하면 끝입니다.

---

## 📜 Declarative vs Scripted Pipeline

Jenkins 파이프라인 문법에는 두 가지 사투리가 있습니다.

1.  **Scripted Pipeline:** Groovy 언어를 그대로 씁니다. 자유도가 높지만 어렵고 복잡합니다. (옛날 방식)
2.  **Declarative Pipeline:** 정해진 구조(Structure)를 따릅니다. 읽기 쉽고 깔끔하며, Jenkins가 문법 체크도 해줍니다. **(모던 방식, 강력 추천)**

우리는 **Declarative Pipeline**만 다룹니다. 이것이 업계 표준입니다.

---

## 🏗️ Core Syntax (핵심 문법)

`Jenkinsfile`의 뼈대는 다음과 같습니다.

```groovy
pipeline {
    agent any  // 어디서 실행할 것인가? (일단 아무 데서나)
    
    stages {   // 작업의 단계들
        stage('Build') { // 1단계: 빌드
            steps {
                echo 'Building...' // 실행할 명령어
            }
        }
        stage('Test') {  // 2단계: 테스트
            steps {
                echo 'Testing...'
            }
        }
    }
}
```

*   **`pipeline { ... }`**: 전체 파이프라인의 시작과 끝입니다.
*   **`agent`**: 작업을 수행할 노드를 지정합니다.
    *   `agent any`: 빈 일꾼 아무나 나와라.
    *   `agent { docker { image 'node:18' } }`: **(Big Tech Best Practice)** 노드(Node)나 파이썬(Python) 등 특정 환경이 세팅된 도커 컨테이너를 일회용으로 띄워서 실행하고 버립니다. 환경 오염을 막는 최고의 방법입니다.
*   **`stages`**: 전체 프로세스를 정의하는 단계의 묶음입니다.
*   **`stage`**: 개별 단계(Build, Test, Deploy 등)입니다. Jenkins UI에서 시각적으로 나뉘어 보입니다.
*   **`steps`**: 실제 실행할 명령어(`sh`, `echo`, `git` 등)를 적는 곳입니다.

---

## 🛠️ 실습: Your First Pipeline

이제 진짜 파이프라인을 만들어 봅시다.

### 1. Jenkinsfile 작성
여러분의 로컬 컴퓨터(또는 GitHub 저장소)의 프로젝트 루트에 `Jenkinsfile` (확장자 없음)을 만들고 아래 내용을 붙여넣으세요.

```groovy
pipeline {
    agent any

    stages {
        stage('Compile') {
            steps {
                echo 'Compiling the code...'
                sh 'echo "Compile Complete" > build.log'
            }
        }
        stage('Unit Test') {
            steps {
                echo 'Running unit tests...'
                // 실제로는 ./mvnw test 또는 npm test 등이 들어갑니다.
            }
        }
        stage('Package') {
            steps {
                echo 'Packaging application...'
            }
        }
    }
}
```

### 2. Jenkins Job 생성
1.  Jenkins > **New Item** 클릭.
2.  이름: `MyFirstPipeline`, 유형: **Pipeline** 선택. OK.
3.  **Pipeline** 섹션으로 이동.
4.  **Definition**을 `Pipeline script from SCM`으로 변경. (SCM은 Source Code Management, 즉 Git을 말합니다.)
5.  **SCM**을 `Git`으로 선택하고 여러분의 Repository URL을 입력합니다.
6.  **Script Path**는 `Jenkinsfile` 그대로 둡니다.
7.  **Save**.

### 3. 실행 (Build Now)
**Build Now**를 누르세요.
잠시 후, "Stage View"라는 멋진 시각화 화면이 뜰 것입니다. 각 단계가 성공할 때마다 초록색 박스가 채워지는 쾌감을 느껴보세요. 이것이 바로 파이프라인입니다.

---

## 🛡️ Best Practices

1.  **Fail Fast (빨리 실패하라):** 시간이 오래 걸리는 작업보다, 빨리 검증할 수 있는 작업을 앞에 배치하세요. (문법 검사 -> 유닛 테스트 -> 통합 테스트 순서)
2.  **Logical Stages:** 단계를 명확히 나누세요. 그래야 어디서 실패했는지 한눈에 알 수 있습니다.

---

## 📝 3주차 과제: Post Actions

파이프라인이 끝나면 결과를 알려줘야겠죠?
`post` 섹션을 추가하여 빌드 결과에 따라 다른 메시지를 출력하도록 수정해 보세요.

*   성공 시(success): "Build Succeeded! 🚀" 출력
*   실패 시(failure): "Build Failed... 😭" 출력

**힌트:**
```groovy
pipeline {
    ...
    stages { ... }
    
    post {
        success {
            echo 'Good job!'
        }
        failure {
            echo 'Oh no...'
        }
    }
}
```

---

## ✅ Self-Assessment Quiz

1.  **Q1.** GUI로 설정을 관리하는 것보다 코드로 관리하는 것의 장점은 무엇인가요? (2가지 이상)
2.  **Q2.** Jenkins 파이프라인 문법 중, 우리가 사용하는 현대적이고 구조화된 문법의 이름은 무엇인가요?
3.  **Q3.** 파이프라인의 각 단계(`stage`) 안에 실제 명령어를 적는 블록의 이름은 무엇인가요?

---

다음 주에는 파이프라인에 **지능(Logic)**을 불어넣습니다. 조건문, 반복문, 그리고 병렬 실행을 통해 시간을 획기적으로 단축하는 고급 기술을 배웁니다.

**Code your infrastructure.**
