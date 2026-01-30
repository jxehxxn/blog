---
layout: post
title: "DevSecOps Mastery: 4주차 - 고급 파이프라인 기법과 최적화"
---

단순한 일자형 파이프라인(Build -> Test -> Deploy)은 이제 졸업할 때가 되었습니다.
현실 세계의 배포 과정은 훨씬 복잡합니다.
"테스트가 실패하면 배포하지 마라", "메인 브랜치일 때만 배포해라", "테스트 10개를 동시에 돌려라"...

오늘은 Jenkins 파이프라인을 더 똑똑하고(Smart), 더 빠르게(Fast) 만드는 고급 기법들을 전수해 드리겠습니다.

---

## 🌍 Environment Variables (환경 변수)

파이프라인 안에서 변수를 사용하면 유연성이 높아집니다.
`environment { ... }` 블록을 사용합니다.

```groovy
pipeline {
    agent any
    environment {
        APP_VERSION = '1.0.0'
        DEPLOY_TARGET = 'production-server-01'
        // Jenkins Credentials에 저장된 비밀키를 안전하게 불러오기
        AWS_KEY = credentials('my-aws-access-key') 
    }
    stages {
        stage('Print Env') {
            steps {
                echo "Deploying version ${APP_VERSION} to ${DEPLOY_TARGET}"
                // 비밀키는 로그에 **** 로 마스킹되어 출력됩니다. (보안!)
                sh 'echo using key $AWS_KEY' 
            }
        }
    }
}
```

---

## 🚦 Flow Control (흐름 제어)

무조건 실행하는 게 아니라, 상황에 따라 판단하게 만듭니다.

### 1. `when` Directive (조건부 실행)
"이 단계는 `main` 브랜치일 때만 실행해!"

```groovy
stage('Deploy to Production') {
    when {
        branch 'main'
    }
    steps {
        echo 'Deploying to Master Server...'
    }
}
```
개발(dev) 브랜치에서는 이 단계가 **Skipped(건너뜀)** 처리됩니다. 안전한 배포를 위한 핵심입니다.

### 2. `timeout` (시간 제한)
빌드가 멈춰서(Hang) 무한히 돌아가는 것을 방지합니다. 빅테크 기업들은 모든 파이프라인에 타임아웃을 겁니다. 자원은 돈이니까요.

```groovy
options {
    timeout(time: 10, unit: 'MINUTES') // 10분 넘으면 강제 종료
}
```

---

## ⚡ Speeding Up: Parallel Execution (병렬 실행)

순차적으로 실행하면 시간이 오래 걸립니다. 서로 의존성이 없는 작업은 **동시에** 실행하는 것이 국룰입니다.
예를 들어, `Unit Test`(기능 테스트)와 `Linting`(코드 스타일 검사)은 서로 기다릴 필요가 없습니다.

```groovy
stage('Quality Checks') {
    parallel {
        stage('Unit Tests') {
            steps { echo 'Running tests...' }
        }
        stage('Linting') {
            steps { echo 'Checking style...' }
        }
        stage('Security Scan') {
            steps { echo 'Scanning vulnerabilities...' }
        }
    }
}
```
이렇게 하면 3가지 작업이 동시에 시작되어, 가장 오래 걸리는 작업 시간만큼만 소요됩니다. 전체 빌드 시간을 획기적으로 줄일 수 있습니다.

---

## 📚 Don't Repeat Yourself (DRY) - Shared Libraries

(개념 소개)
회사의 프로젝트가 100개라고 해봅시다. 100개의 Jenkinsfile을 매번 처음부터 다시 짤까요?
아니요. **Shared Library(공유 라이브러리)**를 사용합니다.

공통된 로직(예: 표준 배포 스크립트, 보안 스캔 스크립트)을 별도의 Git 저장소에 모아두고, Jenkinsfile에서는 `import`해서 함수처럼 가져다 쓰는 방식입니다.
이것이 대규모 DevSecOps를 지탱하는 기둥입니다. (자세한 구현은 심화 과정에서 다룹니다.)

---

## 🛠️ 실습: Optimizing the Pipeline

지난주에 만든 Jenkinsfile을 업그레이드해 봅시다.

### 목표
1.  **병렬 처리:** `Build` 후에 `Test`와 `Lint`를 동시에 실행하세요.
2.  **조건부 배포:** `Deploy` 단계는 오직 `main` 브랜치에서만 실행되도록 하세요.
3.  **타임아웃:** 전체 파이프라인이 5분을 넘기지 않도록 설정하세요.

### 모범 답안 구조

```groovy
pipeline {
    agent any
    options {
        timeout(time: 5, unit: 'MINUTES')
    }
    stages {
        stage('Build') {
            steps { echo 'Building...' }
        }
        
        stage('Quality Check') {
            parallel {
                stage('Test') { steps { echo 'Testing...' } }
                stage('Lint') { steps { echo 'Linting...' } }
            }
        }
        
        stage('Deploy') {
            when { branch 'main' }
            steps { echo 'Deploying to Prod...' }
        }
    }
}
```

---

## 📝 4주차 과제: 커스텀 환경 변수 활용

1.  `pipeline` 상단에 `TEAM_NAME`이라는 환경 변수를 정의하고 여러분의 팀 이름(또는 닉네임)을 넣으세요.
2.  `Deploy` 단계에서 "Deployed by ${TEAM_NAME}" 이라는 메시지가 출력되도록 수정하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 파이프라인의 특정 단계(`stage`)를 특정 조건(예: 브랜치 이름)에서만 실행하게 해주는 지시어는 무엇인가요?
2.  **Q2.** 여러 단계를 동시에 실행하여 전체 빌드 시간을 단축시키는 블록의 이름은 무엇인가요?
3.  **Q3.** Jenkinsfile 내에서 비밀번호와 같은 민감한 정보를 안전하게 사용하기 위해 `environment` 블록 안에서 사용하는 함수는 무엇인가요?

---

다음 주는 **Docker**와 파이프라인이 만나는 순간입니다.
단순히 Docker를 설치하는 것을 넘어, **"도커 안에서 도커를 사용하는(Docker in Docker)"** 마법과 컨테이너 기반의 동적 에이전트 활용법을 배웁니다. 기대하셔도 좋습니다.

**Optimization is key.**
