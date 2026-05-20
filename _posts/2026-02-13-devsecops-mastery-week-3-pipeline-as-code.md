---
layout: post
title: "DevSecOps Mastery 3주차 - 자동화의 심장, Pipeline as Code"
date: 2026-05-20 11:30:00 +0900
---

# DevSecOps Mastery 3주차 - 자동화의 심장, Pipeline as Code

## 수동 설정의 한계, 그리고 코드로 다루는 인프라

지금까지 우리는 Jenkins 웹 화면에서 버튼을 눌러가며(이른바 GUI 클릭 조작) 파이프라인을 구성해 왔다. GUI 기반 방식은 직관적이다. 그러나 엔터프라이즈 환경으로 들어가면 곧 한계를 드러낸다.

### 한 사고에서 시작하는 이야기

2023년에 있었던 일이다. 한 핀테크 회사에서 핵심 CI/CD 서버가 하드웨어 장애로 통째로 사라졌다. 100여 개 마이크로서비스의 복잡한 빌드 설정이 한순간에 증발했고 DevOps 팀은 3주 동안 설정을 수동으로 복구해야 했다. 그 사이 신규 배포는 완전히 멈췄고 긴급 패치마저 손으로 직접 진행해야 했다.

Pipeline as Code를 적용했다면 이런 재난은 처음부터 일어나지 않는다.

## 1. ClickOps의 한계 — 엔터프라이즈 관점

### 1.1 추적성 부재

GUI 기반 설정은 변경 이력을 남기지 않는다. 실제 DevOps 환경에서 어떤 문제가 생기는지 보자.

사례 연구: Netflix의 초기 CI/CD 문제
Netflix는 2010년 무렵 수백 개 마이크로서비스를 GUI로 관리하다가 벽에 부딪혔다.
- 배포가 실패해도 누가, 언제, 무엇을 바꿨는지 추적이 안 됨
- 이전 설정으로 되돌릴 방법이 없음
- 설정 한 줄 바꿨는데 의도치 않은 사이드 이펙트 발생

Netflix는 이 문제를 풀려고 오픈소스 배포 플랫폼 Spinnaker를 만들었고, 모든 설정을 코드로 옮겼다.

### 1.2 규모를 키울 수 없는 구조

```bash
# 엔터프라이즈 환경의 일반적인 규모
마이크로서비스: 200~500개
환경: DEV, QA, STAGING, PROD (4개)
총 파이프라인: 800~2,000개

# GUI로 관리할 경우의 작업 시간
파이프라인 하나당 설정 시간: 30분
전체 설정 시간: 400~1,000시간 (25~62주)
```

### 1.3 일관성이 깨진다

수동 설정은 사람의 실수를 부른다.
- 환경마다 설정이 미묘하게 다르다
- 팀원마다 손에 익은 방식이 다르다
- 보안 설정이 빠진다

### 1.4 재해 복구가 불가능

실제 사례: GitLab.com 데이터 삭제 사고 (2017)
GitLab의 데이터베이스 관리자가 실수로 프로덕션 DB를 삭제한 사건이다. 이때 백업 시스템이 줄줄이 무너졌는데, Infrastructure as Code로 관리되던 부분은 빠르게 살아났다. 반면 GUI로 만진 부분은 처음부터 손으로 다시 짜야 했다.

## 2. Pipeline as Code, 현대 DevOps의 핵심

### 2.1 Jenkinsfile의 철학

Jenkinsfile은 그냥 설정 파일이 아니다. Infrastructure as Code(IaC) 철학을 그대로 구현한 결과물이다.

```groovy
// 이것은 단순한 스크립트가 아닌 인프라 정의서입니다
pipeline {
    // 실행 환경 정의 (Infrastructure)
    agent { 
        kubernetes {
            yaml '''
              apiVersion: v1
              kind: Pod
              spec:
                containers:
                - name: gradle
                  image: gradle:7.6-jdk11
                  command: ['cat']
                  tty: true
            '''
        }
    }
    
    // 보안 정책 정의
    options {
        skipDefaultCheckout()
        timeout(time: 1, unit: 'HOURS')
        retry(3)
    }
    
    // 프로세스 정의
    stages {
        stage('Security Scan') {
            parallel {
                stage('SAST') {
                    steps {
                        container('gradle') {
                            sh './gradlew sonarqube'
                        }
                    }
                }
                stage('Dependency Check') {
                    steps {
                        sh './gradlew dependencyCheckAnalyze'
                    }
                }
            }
        }
    }
}
```

### 2.2 버전 관리, 무엇이 달라지는가

GUI 설정 시절에는 변경 사항을 추적할 수 없었다. 롤백도, 브랜치별 다른 빌드 설정도 불가능했다.

Jenkinsfile로 옮긴 뒤에는 이야기가 달라진다. `git blame`으로 누가 바꿨는지 본다. `git revert`로 즉시 되돌린다. 브랜치마다 독립된 파이프라인을 둘 수 있다.

```bash
# 파이프라인 변경 이력 추적
git log --oneline -- Jenkinsfile
a1b2c3d feat: add security scanning stage
d4e5f6g fix: maven memory settings
g7h8i9j refactor: parallel test execution

# 특정 변경사항 확인
git show a1b2c3d

# 문제 발생 시 즉시 롤백
git revert a1b2c3d
```

### 2.3 코드 재사용과 템플릿화

Shared Libraries로 만드는 엔터프라이즈 표준

```groovy
// vars/standardPipeline.groovy (공통 라이브러리)
def call(Map config) {
    pipeline {
        agent any
        stages {
            stage('Security') {
                steps {
                    securityScan(config.language)
                }
            }
            stage('Build') {
                steps {
                    buildApplication(config.buildTool)
                }
            }
            stage('Test') {
                parallel {
                    stage('Unit Tests') {
                        steps {
                            runUnitTests(config.testFramework)
                        }
                    }
                    stage('Integration Tests') {
                        steps {
                            runIntegrationTests(config.testFramework)
                        }
                    }
                }
            }
        }
    }
}
```

```groovy
// 프로젝트별 Jenkinsfile
@Library('shared-pipeline-library') _

standardPipeline([
    language: 'java',
    buildTool: 'gradle',
    testFramework: 'junit5'
])
```

## 3. Declarative와 Scripted, 어느 쪽을 고를 것인가

### 3.1 두 방식의 차이

Scripted Pipeline은 옛 방식이다.
```groovy
node('master') {
    try {
        stage('Checkout') {
            checkout scm
        }
        stage('Build') {
            if (env.BRANCH_NAME == 'master') {
                sh 'mvn clean package'
            } else {
                sh 'mvn clean compile'
            }
        }
    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        cleanWs()
    }
}
```

Declarative Pipeline이 지금의 표준이다.
```groovy
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            when {
                branch 'master'
            }
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Build Non-Master') {
            when {
                not { branch 'master' }
            }
            steps {
                sh 'mvn clean compile'
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
```

### 3.2 엔터프라이즈에서 무엇을 고를까

Declarative Pipeline을 골라야 하는 이유는 네 가지다.

1. 린팅 지원. 문법 오류를 미리 잡는다.
```bash
# Jenkins CLI를 통한 문법 검증
java -jar jenkins-cli.jar -s http://localhost:8080 \
    declarative-linter < Jenkinsfile
```

2. Blue Ocean 통합. 시각적 파이프라인 편집기를 쓸 수 있다.
3. 표준화. HashiCorp Terraform처럼 선언적 구문이 업계 표준으로 자리잡고 있다.
4. 보안. 제한된 구문이라 악성 코드를 끼워 넣기 어렵다.

### 3.3 실제 기업의 채택 사례

Google의 접근법
Google은 사내 CI/CD 시스템을 완전히 선언적 방식으로 통일했다. 이유는 셋이다.
- 1만 명이 넘는 개발자가 같은 방식으로 파이프라인을 짠다
- 자동 보안 스캔과 컴플라이언스 체크가 가능하다
- 머신러닝 기반 빌드 최적화를 얹기 쉽다

### 3.4 마이그레이션 전략

레거시 Scripted를 Declarative로 옮기는 방법

```groovy
// 기존 Scripted Pipeline
node {
    stage('Build') {
        sh 'echo "Building..."'
    }
    stage('Test') {
        parallel(
            'Unit': {
                sh 'echo "Unit tests"'
            },
            'Integration': {
                sh 'echo "Integration tests"'
            }
        )
    }
}

// Declarative로 변환
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'echo "Building..."'
            }
        }
        stage('Test') {
            parallel {
                stage('Unit') {
                    steps {
                        sh 'echo "Unit tests"'
                    }
                }
                stage('Integration') {
                    steps {
                        sh 'echo "Integration tests"'
                    }
                }
            }
        }
    }
}
```

## 4. Declarative Pipeline 핵심 문법

### 4.1 Pipeline 블록 — 모든 것은 여기서 시작한다

```groovy
pipeline {
    // 모든 설정과 정의가 이 안에 포함됩니다
}
```

이 블록이 있어야 Jenkins가 Declarative Pipeline으로 인식한다. Groovy 문법을 따르지만 제한된 DSL(Domain Specific Language)이다.

### 4.2 Agent — 실행 환경을 어디에 둘 것인가

#### 4.2.1 기본 Agent 유형

Static Agent (전통적 방식)
```groovy
agent {
    label 'linux && x64'  // 라벨 기반 노드 선택
}

agent {
    node {
        label 'master'
        customWorkspace '/opt/jenkins/workspace/custom'
    }
}
```

Dynamic Agent (Container 기반 — 권장)
```groovy
agent {
    docker {
        image 'maven:3.8.6-openjdk-11'
        args '-v /var/run/docker.sock:/var/run/docker.sock'
        reuseNode true  // 같은 노드에서 모든 스테이지 실행
    }
}

agent {
    kubernetes {
        yaml '''
          apiVersion: v1
          kind: Pod
          metadata:
            name: jenkins-agent
          spec:
            serviceAccountName: jenkins
            containers:
            - name: maven
              image: maven:3.8.6-openjdk-11
              command: ['cat']
              tty: true
              resources:
                requests:
                  memory: "1Gi"
                  cpu: "500m"
                limits:
                  memory: "2Gi"
                  cpu: "1000m"
            - name: docker
              image: docker:20.10.17-dind
              securityContext:
                privileged: true
              volumeMounts:
              - name: docker-sock
                mountPath: /var/run/docker.sock
            volumes:
            - name: docker-sock
              hostPath:
                path: /var/run/docker.sock
        '''
    }
}
```

#### 4.2.2 실제 기업의 Agent 전략

Netflix는 완전 컨테이너화로 간다.
```groovy
agent {
    kubernetes {
        // 각 빌드마다 완전히 격리된 환경 제공
        // 보안, 재현성, 확장성 확보
    }
}
```

GitLab은 Runner 기반 분산으로 간다.
```groovy
agent {
    label 'docker-executor'
    // Docker Machine을 통한 동적 스케일링
}
```

### 4.3 Stages와 Stage — 워크플로우 정의

```groovy
stages {
    stage('Pre-build') {
        parallel {
            stage('Security Scan') {
                steps {
                    sh 'trivy fs .'
                }
            }
            stage('Code Quality') {
                steps {
                    sh 'sonar-scanner'
                }
            }
            stage('License Check') {
                steps {
                    sh 'license-check'
                }
            }
        }
    }
    
    stage('Build') {
        when {
            anyOf {
                branch 'develop'
                branch 'master'
                changeRequest()
            }
        }
        steps {
            container('maven') {
                sh 'mvn clean compile'
            }
        }
        post {
            always {
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'target/site',
                    reportFiles: 'index.html',
                    reportName: 'Build Report'
                ])
            }
        }
    }
    
    stage('Test') {
        matrix {
            axes {
                axis {
                    name 'JAVA_VERSION'
                    values '11', '17', '21'
                }
                axis {
                    name 'OS'
                    values 'ubuntu', 'alpine'
                }
            }
            stages {
                stage('Matrix Test') {
                    steps {
                        script {
                            sh "docker run openjdk:${JAVA_VERSION}-${OS} mvn test"
                        }
                    }
                }
            }
        }
    }
}
```

### 4.4 Options — 파이프라인 동작 제어

```groovy
options {
    // 타임아웃 설정 (무한 대기 방지)
    timeout(time: 2, unit: 'HOURS')
    
    // 재시도 설정 (네트워크 장애 등 대응)
    retry(3)
    
    // 빌드 히스토리 관리
    buildDiscarder(logRotator(
        numToKeepStr: '10',
        daysToKeepStr: '30'
    ))
    
    // 체크아웃 스킵 (커스텀 체크아웃 로직 사용 시)
    skipDefaultCheckout()
    
    // 병렬 빌드 방지
    disableConcurrentBuilds()
    
    // 타임스탬프 추가
    timestamps()
    
    // ANSI 색상 출력
    ansiColor('xterm')
    
    // 빌드 실패 시 즉시 중단
    skipStagesAfterUnstable()
}
```

### 4.5 Environment — 환경 변수 관리

```groovy
environment {
    // 전역 환경 변수
    APPLICATION_NAME = 'my-microservice'
    BUILD_VERSION = "${BUILD_NUMBER}-${GIT_COMMIT.take(8)}"
    
    // 조건부 환경 변수
    DEPLOY_ENV = "${env.BRANCH_NAME == 'master' ? 'production' : 'staging'}"
    
    // 보안 자격 증명
    DOCKER_CREDENTIALS = credentials('docker-hub-credentials')
    SONAR_TOKEN = credentials('sonar-token')
    
    // 동적 환경 변수
    TIMESTAMP = sh(
        script: 'date +%Y%m%d-%H%M%S',
        returnStdout: true
    ).trim()
    
    // 플랫폼별 설정
    MAVEN_OPTS = '-Xmx2g -XX:MaxPermSize=512m'
    JAVA_TOOL_OPTIONS = '-Dfile.encoding=UTF-8'
}
```

### 4.6 When — 조건부 실행 로직

```groovy
stage('Deploy to Production') {
    when {
        allOf {
            branch 'master'
            not { changeRequest() }
            expression { 
                return currentBuild.currentResult == 'SUCCESS' 
            }
            environment name: 'DEPLOY_ENV', value: 'production'
        }
    }
    steps {
        sh 'kubectl apply -f k8s-prod/'
    }
}

stage('Security Scan') {
    when {
        anyOf {
            changeset "**/*.java"
            changeset "**/pom.xml"
            triggeredBy 'TimerTrigger'
        }
    }
    steps {
        sh 'owasp-dependency-check'
    }
}
```

### 4.7 Tools — 도구 버전 관리

```groovy
tools {
    maven 'maven-3.8.6'
    jdk 'openjdk-11'
    nodejs 'nodejs-18'
    terraform 'terraform-1.3.0'
}
```

## 5. 실전 Pipeline 구현 — Step-by-Step 가이드

### 5.1 프로젝트 준비

실습용 Spring Boot 프로젝트를 만들어 보자.

```bash
# 프로젝트 디렉토리 생성
mkdir devsecops-pipeline-demo
cd devsecops-pipeline-demo

# Spring Boot 프로젝트 초기화
curl https://start.spring.io/starter.tgz \
    -d dependencies=web,actuator,security \
    -d javaVersion=11 \
    -d bootVersion=2.7.5 \
    -d groupId=com.example \
    -d artifactId=pipeline-demo \
    | tar -xzv --strip-components=1

# Git 저장소 초기화
git init
git add .
git commit -m "Initial Spring Boot project"
```

### 5.2 단계별 Jenkinsfile 구성

#### 5.2.1 Level 1 — 기본 파이프라인

```groovy
// Jenkinsfile
pipeline {
    agent {
        docker {
            image 'openjdk:11-jdk-slim'
            args '-v $HOME/.m2:/root/.m2'  // Maven 캐시 공유
        }
    }
    
    environment {
        APP_NAME = 'pipeline-demo'
        APP_VERSION = "${BUILD_NUMBER}"
        MAVEN_OPTS = '-Dmaven.test.failure.ignore=false'
    }
    
    stages {
        stage('Environment Check') {
            steps {
                sh '''
                    echo "=== Environment Information ==="
                    java -version
                    echo "Maven version: $(mvn -version | head -1)"
                    echo "Workspace: ${WORKSPACE}"
                    echo "Build Number: ${BUILD_NUMBER}"
                    echo "Git Commit: ${GIT_COMMIT}"
                '''
            }
        }
        
        stage('Checkout & Dependencies') {
            steps {
                echo 'Checking out source code...'
                checkout scm
                
                echo 'Downloading dependencies...'
                sh '''
                    chmod +x mvnw
                    ./mvnw dependency:resolve -B
                '''
            }
        }
        
        stage('Compile') {
            steps {
                echo 'Compiling source code...'
                sh './mvnw clean compile -B'
            }
            post {
                always {
                    echo 'Archiving compilation logs...'
                    sh 'find target -name "*.class" | wc -l > compilation-stats.txt'
                }
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        echo 'Running unit tests...'
                        sh '''
                            ./mvnw test -B \
                                -Dtest.reporter=junit \
                                -Dmaven.test.failure.ignore=false
                        '''
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'target/surefire-reports',
                                reportFiles: '*.html',
                                reportName: 'Unit Test Report'
                            ])
                        }
                    }
                }
                
                stage('Integration Tests') {
                    steps {
                        echo 'Running integration tests...'
                        sh './mvnw verify -B -DskipUnitTests=true'
                    }
                }
            }
        }
        
        stage('Package') {
            steps {
                echo 'Creating JAR package...'
                sh '''
                    ./mvnw package -B -DskipTests=true
                    ls -la target/*.jar
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
            cleanWs()
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

#### 5.2.2 Level 2 — 보안과 품질 게이트 추가

```groovy
pipeline {
    agent none
    
    environment {
        SONAR_TOKEN = credentials('sonar-token')
        DOCKER_REGISTRY = 'harbor.company.com'
        IMAGE_NAME = "${DOCKER_REGISTRY}/devsecops/${APP_NAME}"
    }
    
    stages {
        stage('Security & Quality Gates') {
            parallel {
                stage('SAST Scan') {
                    agent { docker { image 'sonarsource/sonar-scanner-cli:latest' } }
                    steps {
                        echo 'Running Static Application Security Testing...'
                        sh '''
                            sonar-scanner \
                                -Dsonar.projectKey=${APP_NAME} \
                                -Dsonar.sources=src/main \
                                -Dsonar.tests=src/test \
                                -Dsonar.java.binaries=target/classes \
                                -Dsonar.host.url=${SONAR_URL} \
                                -Dsonar.login=${SONAR_TOKEN}
                        '''
                    }
                }
                
                stage('Dependency Check') {
                    agent { docker { image 'owasp/dependency-check:latest' } }
                    steps {
                        echo 'Checking for vulnerable dependencies...'
                        sh '''
                            dependency-check.sh \
                                --project "${APP_NAME}" \
                                --scan . \
                                --format HTML \
                                --format JSON \
                                --out dependency-check-report
                        '''
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'dependency-check-report',
                                reportFiles: 'dependency-check-report.html',
                                reportName: 'Dependency Check Report'
                            ])
                        }
                    }
                }
                
                stage('Secrets Scan') {
                    agent { docker { image 'trufflesecurity/trufflehog:latest' } }
                    steps {
                        echo 'Scanning for secrets in code...'
                        sh '''
                            trufflehog filesystem . \
                                --json \
                                --no-update \
                                > secrets-scan-results.json || true
                            
                            # Check if any secrets were found
                            if [ -s secrets-scan-results.json ]; then
                                echo "Potential secrets found!"
                                cat secrets-scan-results.json
                                exit 1
                            else
                                echo "No secrets detected"
                            fi
                        '''
                    }
                }
            }
        }
        
        stage('Build & Test') {
            agent {
                kubernetes {
                    yaml '''
                      apiVersion: v1
                      kind: Pod
                      spec:
                        containers:
                        - name: maven
                          image: maven:3.8.6-openjdk-11
                          command: ['cat']
                          tty: true
                          volumeMounts:
                          - name: maven-cache
                            mountPath: /root/.m2
                        - name: docker
                          image: docker:20.10.17-dind
                          securityContext:
                            privileged: true
                          volumeMounts:
                          - name: docker-sock
                            mountPath: /var/run/docker.sock
                        volumes:
                        - name: maven-cache
                          persistentVolumeClaim:
                            claimName: maven-cache
                        - name: docker-sock
                          hostPath:
                            path: /var/run/docker.sock
                    '''
                }
            }
            stages {
                stage('Build Application') {
                    steps {
                        container('maven') {
                            sh '''
                                ./mvnw clean package -B \
                                    -Dmaven.test.skip=false \
                                    -Dspring.profiles.active=ci
                            '''
                        }
                    }
                }
                
                stage('Build Docker Image') {
                    when {
                        anyOf {
                            branch 'develop'
                            branch 'master'
                        }
                    }
                    steps {
                        container('docker') {
                            script {
                                def image = docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")
                                docker.withRegistry("https://${DOCKER_REGISTRY}", 'harbor-credentials') {
                                    image.push()
                                    image.push('latest')
                                }
                            }
                        }
                    }
                }
            }
        }
        
        stage('Image Security Scan') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'master'
                }
            }
            agent { docker { image 'aquasec/trivy:latest' } }
            steps {
                echo 'Scanning Docker image for vulnerabilities...'
                sh '''
                    trivy image \
                        --format json \
                        --output image-scan-results.json \
                        ${IMAGE_NAME}:${BUILD_NUMBER}
                    
                    # Check for HIGH or CRITICAL vulnerabilities
                    HIGH_VULNS=$(jq '.Results[]?.Vulnerabilities[]? | select(.Severity=="HIGH")' image-scan-results.json | jq -s length)
                    CRITICAL_VULNS=$(jq '.Results[]?.Vulnerabilities[]? | select(.Severity=="CRITICAL")' image-scan-results.json | jq -s length)
                    
                    echo "HIGH vulnerabilities: $HIGH_VULNS"
                    echo "CRITICAL vulnerabilities: $CRITICAL_VULNS"
                    
                    if [ "$CRITICAL_VULNS" -gt 0 ]; then
                        echo "Critical vulnerabilities found! Failing build."
                        exit 1
                    elif [ "$HIGH_VULNS" -gt 5 ]; then
                        echo "Too many high-severity vulnerabilities!"
                        exit 1
                    else
                        echo "Image security scan passed"
                    fi
                '''
            }
        }
    }
}
```

### 5.3 Jenkins 설정 — 엔터프라이즈 수준 구성

#### 5.3.1 Multibranch Pipeline 설정

1. Jenkins 대시보드 → New Item
2. 이름: `devsecops-pipeline-demo`
3. 유형: Multibranch Pipeline 선택
4. Branch Sources → Add source → Git
5. Project Repository: `https://github.com/your-org/devsecops-pipeline-demo`
6. Credentials: GitHub access token 설정
7. Behaviours → Add → Filter by name (with wildcards)
   - Include: `master develop feature/* hotfix/*`
   - Exclude: `experimental/*`
8. Build Configuration → Script Path: `Jenkinsfile`
9. Scan Multibranch Pipeline Triggers
   - Periodically if not otherwise run: `1 minute`
   - Build Configuration → Suppress automatic SCM triggering: disabled

#### 5.3.2 Webhook 설정 (실시간 빌드 트리거)

GitHub Webhook 설정
```bash
# GitHub 저장소 설정
curl -X POST \
  -H "Accept: application/vnd.github.v3+json" \
  -H "Authorization: token ${GITHUB_TOKEN}" \
  https://api.github.com/repos/your-org/devsecops-pipeline-demo/hooks \
  -d '{
    "name": "web",
    "active": true,
    "events": ["push", "pull_request"],
    "config": {
      "url": "https://jenkins.company.com/github-webhook/",
      "content_type": "json"
    }
  }'
```

Jenkins Webhook 플러그인 설정
1. Manage Jenkins → Configure System
2. GitHub 섹션에서 GitHub Servers 추가
3. API URL: `https://api.github.com`
4. Credentials: GitHub Personal Access Token
5. Test connection 클릭하여 연결 확인

### 5.4 실행과 모니터링

#### 5.4.1 파이프라인 실행

```bash
# 코드 변경 및 푸시
echo "// Updated application" >> src/main/java/com/example/Application.java
git add .
git commit -m "feat: add application update"
git push origin develop
```

#### 5.4.2 Blue Ocean을 통한 시각적 모니터링

1. Jenkins 플러그인 설치: Blue Ocean
```bash
# Jenkins CLI로 플러그인 설치
java -jar jenkins-cli.jar -s http://localhost:8080 install-plugin blueocean
```

2. Blue Ocean 접속: `http://localhost:8080/blue`
3. 파이프라인 시각화: 각 스테이지의 실시간 진행상황 확인

#### 5.4.3 파이프라인 메트릭 수집

```groovy
// Jenkinsfile에 메트릭 수집 추가
post {
    always {
        script {
            // 빌드 시간 메트릭
            def buildDuration = currentBuild.duration
            def buildResult = currentBuild.currentResult
            
            // InfluxDB로 메트릭 전송
            sh """
                curl -X POST 'http://influxdb:8086/write?db=jenkins' \
                    --data-binary '
                        jenkins_build_duration,job=${JOB_NAME},result=${buildResult} value=${buildDuration}
                        jenkins_build_result,job=${JOB_NAME} value=${buildResult == "SUCCESS" ? 1 : 0}
                    '
            """
        }
    }
}

## 6. Best Practices — 엔터프라이즈 수준 파이프라인 설계

### 6.1 Fail Fast — 빠른 피드백 루프

#### 6.1.1 최적화된 스테이지 순서

```groovy
pipeline {
    agent none
    
    stages {
        // FAST: 즉시 실패 가능한 검증 (< 30초)
        stage('Fast Checks') {
            parallel {
                stage('Syntax Check') {
                    agent { docker { image 'openjdk:11-jdk-slim' } }
                    steps {
                        sh './mvnw compile -q -B'
                    }
                }
                stage('Lint Check') {
                    agent { docker { image 'maven:3.8.6-openjdk-11' } }
                    steps {
                        sh '''
                            ./mvnw spotless:check -B
                            ./mvnw checkstyle:check -B
                        '''
                    }
                }
                stage('Security Pre-check') {
                    agent { docker { image 'trufflesecurity/trufflehog:latest' } }
                    steps {
                        sh 'trufflehog filesystem . --no-update --fail'
                    }
                }
            }
        }
        
        // MEDIUM: 유닛 테스트 (< 5분)
        stage('Unit Tests') {
            agent { docker { image 'maven:3.8.6-openjdk-11' } }
            steps {
                sh './mvnw test -B -Dmaven.test.failure.ignore=false'
            }
        }
        
        // SLOW: 무거운 작업들 (> 5분)
        stage('Deep Analysis') {
            parallel {
                stage('Integration Tests') {
                    agent { docker { image 'maven:3.8.6-openjdk-11' } }
                    steps {
                        sh './mvnw verify -B -DskipUnitTests=true'
                    }
                }
                stage('SAST Scan') {
                    agent { docker { image 'sonarsource/sonar-scanner-cli:latest' } }
                    steps {
                        sh 'sonar-scanner'
                    }
                }
                stage('Dependency Check') {
                    agent { docker { image 'owasp/dependency-check:latest' } }
                    steps {
                        sh 'dependency-check.sh --project myapp --scan .'
                    }
                }
            }
        }
    }
}
```

#### 6.1.2 실제 기업의 Fail Fast 전략

Google은 세 단계로 검증을 쪼갠다.
- Pre-submit checks: 1~2분 안에 기본 검증
- Post-submit builds: 10~15분 안에 전체 검증
- Nightly builds: 야간에 심화 보안 스캔

### 6.2 보안 중심 파이프라인 설계

#### 6.2.1 Shift-Left Security 구현

```groovy
pipeline {
    agent none
    
    environment {
        // 보안 도구 설정
        SECURITY_SCAN_THRESHOLD = 'MEDIUM'
        FAIL_ON_CRITICAL = 'true'
    }
    
    stages {
        stage('Security Gates') {
            parallel {
                stage('Secrets Detection') {
                    agent { docker { image 'trufflesecurity/trufflehog:latest' } }
                    steps {
                        sh '''
                            echo "Scanning for secrets..."
                            trufflehog filesystem . \
                                --json \
                                --no-update \
                                --fail \
                                --config=.trufflehog.yml
                        '''
                    }
                }
                
                stage('License Compliance') {
                    agent { docker { image 'fossas/fossa-cli:latest' } }
                    steps {
                        sh '''
                            fossa analyze
                            fossa test --timeout 600
                        '''
                    }
                }
                
                stage('Infrastructure Scan') {
                    agent { docker { image 'bridgecrew/checkov:latest' } }
                    steps {
                        sh '''
                            checkov -d . \
                                --framework dockerfile \
                                --framework kubernetes \
                                --output json \
                                --output-file checkov-report.json
                        '''
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: '.',
                                reportFiles: 'checkov-report.json',
                                reportName: 'Infrastructure Security Scan'
                            ])
                        }
                    }
                }
            }
        }
        
        stage('SAST Analysis') {
            parallel {
                stage('SonarQube SAST') {
                    agent { docker { image 'sonarsource/sonar-scanner-cli:latest' } }
                    environment {
                        SONAR_TOKEN = credentials('sonar-token')
                    }
                    steps {
                        sh '''
                            sonar-scanner \
                                -Dsonar.projectKey=${JOB_NAME} \
                                -Dsonar.sources=src/main \
                                -Dsonar.java.binaries=target/classes \
                                -Dsonar.qualitygate.wait=true \
                                -Dsonar.qualitygate.timeout=300
                        '''
                    }
                }
                
                stage('Semgrep SAST') {
                    agent { docker { image 'returntocorp/semgrep:latest' } }
                    steps {
                        sh '''
                            semgrep \
                                --config=auto \
                                --json \
                                --output semgrep-results.json \
                                .
                            
                            # Check for high-severity findings
                            HIGH_FINDINGS=$(jq '[.results[] | select(.extra.severity == "ERROR")] | length' semgrep-results.json)
                            if [ "$HIGH_FINDINGS" -gt 0 ]; then
                                echo "High-severity security issues found: $HIGH_FINDINGS"
                                jq '.results[] | select(.extra.severity == "ERROR")' semgrep-results.json
                                exit 1
                            fi
                        '''
                    }
                }
            }
        }
    }
}
```

### 6.3 성능 최적화 전략

#### 6.3.1 병렬 처리 최적화

```groovy
pipeline {
    agent none
    
    stages {
        stage('Parallel Build Matrix') {
            matrix {
                axes {
                    axis {
                        name 'JAVA_VERSION'
                        values '11', '17', '21'
                    }
                    axis {
                        name 'PROFILE'
                        values 'dev', 'prod'
                    }
                }
                excludes {
                    exclude {
                        axis {
                            name 'JAVA_VERSION'
                            values '21'
                        }
                        axis {
                            name 'PROFILE'
                            values 'prod'
                        }
                    }
                }
                stages {
                    stage('Build & Test Matrix') {
                        agent {
                            kubernetes {
                                yaml """
                                  apiVersion: v1
                                  kind: Pod
                                  spec:
                                    containers:
                                    - name: maven
                                      image: maven:3.8.6-openjdk-${JAVA_VERSION}
                                      command: ['cat']
                                      tty: true
                                      resources:
                                        requests:
                                          memory: "2Gi"
                                          cpu: "1"
                                        limits:
                                          memory: "4Gi"
                                          cpu: "2"
                                """
                            }
                        }
                        steps {
                            container('maven') {
                                sh '''
                                    ./mvnw clean test -B \
                                        -Dspring.profiles.active=${PROFILE} \
                                        -Dmaven.test.parallel=4
                                '''
                            }
                        }
                    }
                }
            }
        }
    }
}
```

#### 6.3.2 캐시 활용 전략

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
              apiVersion: v1
              kind: Pod
              spec:
                containers:
                - name: maven
                  image: maven:3.8.6-openjdk-11
                  command: ['cat']
                  tty: true
                  volumeMounts:
                  - name: maven-cache
                    mountPath: /root/.m2
                  - name: sonar-cache
                    mountPath: /opt/sonar-scanner/.sonar/cache
                volumes:
                - name: maven-cache
                  persistentVolumeClaim:
                    claimName: maven-cache-pvc
                - name: sonar-cache
                  persistentVolumeClaim:
                    claimName: sonar-cache-pvc
            '''
        }
    }
    
    stages {
        stage('Cache Optimization') {
            steps {
                container('maven') {
                    sh '''
                        # Maven 의존성 캐시 최적화
                        ./mvnw dependency:go-offline -B
                        
                        # 캐시 상태 확인
                        echo "Maven cache size: $(du -sh /root/.m2 | cut -f1)"
                        echo "Cached artifacts: $(find /root/.m2/repository -name "*.jar" | wc -l)"
                    '''
                }
            }
        }
    }
}
```

### 6.4 모니터링과 알림

#### 6.4.1 고급 알림 시스템

```groovy
pipeline {
    agent any
    
    post {
        success {
            script {
                if (env.BRANCH_NAME == 'master') {
                    slackSend(
                        channel: '#deployments',
                        color: 'good',
                        message: """
                            Production deployment successful!
                            
                            *Project*: ${env.JOB_NAME}
                            *Build*: #${env.BUILD_NUMBER}
                            *Duration*: ${currentBuild.durationString}
                            *Commit*: ${env.GIT_COMMIT.take(8)}
                            
                            <${env.BUILD_URL}|View Build>
                        """,
                        teamDomain: 'company-workspace',
                        token: 'slack-token'
                    )
                }
            }
        }
        
        failure {
            script {
                def failedStage = currentBuild.rawBuild.getExecution().getFailedNodes().first()?.displayName ?: 'Unknown'
                
                emailext(
                    to: "${env.CHANGE_AUTHOR_EMAIL}, devops-team@company.com",
                    subject: "Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <h2>Build Failure Notification</h2>
                        
                        <table>
                            <tr><td><strong>Project:</strong></td><td>${env.JOB_NAME}</td></tr>
                            <tr><td><strong>Build Number:</strong></td><td>#${env.BUILD_NUMBER}</td></tr>
                            <tr><td><strong>Failed Stage:</strong></td><td>${failedStage}</td></tr>
                            <tr><td><strong>Branch:</strong></td><td>${env.BRANCH_NAME}</td></tr>
                            <tr><td><strong>Commit:</strong></td><td>${env.GIT_COMMIT}</td></tr>
                            <tr><td><strong>Author:</strong></td><td>${env.CHANGE_AUTHOR}</td></tr>
                        </table>
                        
                        <p><a href="${env.BUILD_URL}console">View Console Output</a></p>
                    """,
                    mimeType: 'text/html'
                )
                
                // PagerDuty 알림 (프로덕션 실패 시)
                if (env.BRANCH_NAME == 'master') {
                    sh '''
                        curl -X POST https://events.pagerduty.com/v2/enqueue \
                            -H "Content-Type: application/json" \
                            -d '{
                                "routing_key": "'"${env.PAGERDUTY_INTEGRATION_KEY}"'",
                                "event_action": "trigger",
                                "payload": {
                                    "summary": "Production build failure: '"${env.JOB_NAME}"'",
                                    "source": "jenkins",
                                    "severity": "critical",
                                    "custom_details": {
                                        "build_number": "'"${env.BUILD_NUMBER}"'",
                                        "failed_stage": "'"${failedStage}"'",
                                        "build_url": "'"${env.BUILD_URL}"'"
                                    }
                                }
                            }'
                    '''
                }
            }
        }
        
        unstable {
            slackSend(
                channel: '#ci-cd',
                color: 'warning',
                message: """
                    Build unstable: ${env.JOB_NAME} #${env.BUILD_NUMBER}
                    Some tests failed but build completed.
                    <${env.BUILD_URL}testReport|View Test Results>
                """
            )
        }
    }
}

## 7. 실전 프로젝트 — 난이도별 실습

### 7.1 초급 실습 — 기본 파이프라인 구축

#### 과제 1 — Node.js 애플리케이션 파이프라인

목표: Express.js 앱을 위한 기본 CI/CD 파이프라인 구축

프로젝트 설정
```bash
# Node.js 프로젝트 생성
mkdir nodejs-pipeline-demo
cd nodejs-pipeline-demo
npm init -y
npm install express
npm install --save-dev jest supertest
```

package.json 스크립트 추가
```json
{
  "scripts": {
    "start": "node app.js",
    "test": "jest",
    "lint": "eslint .",
    "security-check": "npm audit"
  }
}
```

요구사항은 다음과 같다.
1. 코드 린팅 (`eslint`)
2. 유닛 테스트 실행 (`jest`)
3. 보안 취약점 스캔 (`npm audit`)
4. Docker 이미지 빌드
5. 테스트 결과 리포트 생성

Jenkinsfile 템플릿
```groovy
pipeline {
    agent {
        docker {
            image 'node:18-alpine'
            args '-u root:root'
        }
    }
    
    stages {
        stage('Install Dependencies') {
            steps {
                // TODO: npm install 명령어 구현
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('Lint') {
                    steps {
                        // TODO: ESLint 실행
                    }
                }
                stage('Security Check') {
                    steps {
                        // TODO: npm audit 실행
                    }
                }
            }
        }
        
        stage('Test') {
            steps {
                // TODO: Jest 테스트 실행
            }
            post {
                always {
                    // TODO: 테스트 결과 발행
                }
            }
        }
        
        stage('Build') {
            steps {
                // TODO: Docker 이미지 빌드
            }
        }
    }
}
```

정답 예시
```groovy
pipeline {
    agent {
        docker {
            image 'node:18-alpine'
            args '-u root:root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    environment {
        APP_NAME = 'nodejs-demo'
        IMAGE_TAG = "${BUILD_NUMBER}-${GIT_COMMIT.take(8)}"
    }
    
    stages {
        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "Installing dependencies..."
                    npm ci
                    npm ls
                '''
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('Lint') {
                    steps {
                        sh '''
                            echo "Running ESLint..."
                            npx eslint . --format junit --output-file eslint-results.xml || true
                        '''
                    }
                    post {
                        always {
                            publishTestResults testResultsPattern: 'eslint-results.xml'
                        }
                    }
                }
                
                stage('Security Check') {
                    steps {
                        sh '''
                            echo "Running security audit..."
                            npm audit --audit-level=high --json > audit-results.json || true
                            
                            # Check for high/critical vulnerabilities
                            HIGH_VULNS=$(jq '.metadata.vulnerabilities.high // 0' audit-results.json)
                            CRITICAL_VULNS=$(jq '.metadata.vulnerabilities.critical // 0' audit-results.json)
                            
                            echo "High vulnerabilities: $HIGH_VULNS"
                            echo "Critical vulnerabilities: $CRITICAL_VULNS"
                            
                            if [ "$CRITICAL_VULNS" -gt 0 ]; then
                                echo "Critical vulnerabilities found!"
                                exit 1
                            fi
                        '''
                    }
                }
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    echo "Running tests..."
                    npm test -- --ci --testResultsProcessor=jest-junit
                '''
            }
            post {
                always {
                    publishTestResults testResultsPattern: 'junit.xml'
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'coverage',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
        
        stage('Build Docker Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    sh '''
                        echo "Building Docker image..."
                        docker build -t ${APP_NAME}:${IMAGE_TAG} .
                        docker tag ${APP_NAME}:${IMAGE_TAG} ${APP_NAME}:latest
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
        always {
            cleanWs()
        }
    }
}
```

### 7.2 중급 실습 — 마이크로서비스 파이프라인

#### 과제 2 — 멀티 서비스 모노레포 파이프라인

시나리오: 마이크로서비스 셋(api, web, worker)이 한 저장소에 함께 있다.

폴더 구조
```
microservices-monorepo/
├── services/
│   ├── api/
│   │   ├── Dockerfile
│   │   ├── pom.xml
│   │   └── src/
│   ├── web/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── src/
│   └── worker/
│       ├── Dockerfile
│       ├── requirements.txt
│       └── src/
├── k8s/
│   ├── api.yaml
│   ├── web.yaml
│   └── worker.yaml
└── Jenkinsfile
```

요구사항은 다섯이다.
1. 변경된 서비스만 빌드(Smart Detection)
2. 서비스별 병렬 빌드
3. 통합 테스트 실행
4. Kubernetes 배포 매니페스트 검증
5. 단계적 배포 (dev → staging → prod)

고급 Jenkinsfile 예시
```groovy
pipeline {
    agent none
    
    environment {
        REGISTRY = 'harbor.company.com/microservices'
        KUBERNETES_NAMESPACE_DEV = 'microservices-dev'
        KUBERNETES_NAMESPACE_STAGING = 'microservices-staging'
        KUBERNETES_NAMESPACE_PROD = 'microservices-prod'
    }
    
    stages {
        stage('Change Detection') {
            agent { label 'master' }
            steps {
                script {
                    def changedServices = []
                    
                    // Git diff를 통한 변경 감지
                    def changes = sh(
                        script: """
                            if [ "${env.CHANGE_ID}" != "" ]; then
                                git diff --name-only origin/main...HEAD
                            else
                                git diff --name-only HEAD~1 HEAD
                            fi
                        """,
                        returnStdout: true
                    ).trim().split('\n')
                    
                    changes.each { change ->
                        if (change.startsWith('services/api/')) {
                            changedServices.add('api')
                        } else if (change.startsWith('services/web/')) {
                            changedServices.add('web')
                        } else if (change.startsWith('services/worker/')) {
                            changedServices.add('worker')
                        }
                    }
                    
                    changedServices = changedServices.unique()
                    env.CHANGED_SERVICES = changedServices.join(',')
                    
                    echo "Changed services: ${env.CHANGED_SERVICES}"
                }
            }
        }
        
        stage('Build Services') {
            when {
                expression { env.CHANGED_SERVICES != '' }
            }
            matrix {
                axes {
                    axis {
                        name 'SERVICE'
                        values 'api', 'web', 'worker'
                    }
                }
                when {
                    expression { env.CHANGED_SERVICES.split(',').contains(SERVICE) }
                }
                stages {
                    stage('Build Service') {
                        agent {
                            kubernetes {
                                yaml """
                                  apiVersion: v1
                                  kind: Pod
                                  spec:
                                    containers:
                                    - name: builder
                                      image: ${SERVICE == 'api' ? 'maven:3.8.6-openjdk-11' : 
                                              SERVICE == 'web' ? 'node:18-alpine' : 'python:3.9-slim'}
                                      command: ['cat']
                                      tty: true
                                    - name: docker
                                      image: docker:20.10.17-dind
                                      securityContext:
                                        privileged: true
                                """
                            }
                        }
                        stages {
                            stage('Test') {
                                steps {
                                    container('builder') {
                                        script {
                                            dir("services/${SERVICE}") {
                                                if (SERVICE == 'api') {
                                                    sh './mvnw test -B'
                                                } else if (SERVICE == 'web') {
                                                    sh 'npm ci && npm test'
                                                } else if (SERVICE == 'worker') {
                                                    sh 'pip install -r requirements-test.txt && pytest'
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                            
                            stage('Build Image') {
                                steps {
                                    container('docker') {
                                        script {
                                            dir("services/${SERVICE}") {
                                                def image = docker.build("${REGISTRY}/${SERVICE}:${BUILD_NUMBER}")
                                                docker.withRegistry("https://${REGISTRY}", 'harbor-credentials') {
                                                    image.push()
                                                    image.push('latest')
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        
        stage('Integration Tests') {
            when {
                expression { env.CHANGED_SERVICES != '' }
            }
            agent {
                kubernetes {
                    yaml '''
                      apiVersion: v1
                      kind: Pod
                      spec:
                        containers:
                        - name: test-runner
                          image: postman/newman:latest
                          command: ['cat']
                          tty: true
                    '''
                }
            }
            steps {
                container('test-runner') {
                    sh '''
                        echo "Running integration tests..."
                        newman run integration-tests/collection.json \
                            --environment integration-tests/environment.json \
                            --reporters junit,html \
                            --reporter-junit-export newman-results.xml \
                            --reporter-html-export newman-results.html
                    '''
                }
            }
            post {
                always {
                    publishTestResults testResultsPattern: 'newman-results.xml'
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'newman-results.html',
                        reportName: 'Integration Test Report'
                    ])
                }
            }
        }
        
        stage('Deploy') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                }
            }
            stages {
                stage('Deploy to Dev') {
                    agent {
                        kubernetes {
                            yaml '''
                              apiVersion: v1
                              kind: Pod
                              spec:
                                serviceAccountName: jenkins-deployer
                                containers:
                                - name: kubectl
                                  image: bitnami/kubectl:latest
                                  command: ['cat']
                                  tty: true
                            '''
                        }
                    }
                    steps {
                        container('kubectl') {
                            script {
                                env.CHANGED_SERVICES.split(',').each { service ->
                                    sh """
                                        sed 's|{{IMAGE_TAG}}|${BUILD_NUMBER}|g' k8s/${service}.yaml | \\
                                        kubectl apply -n ${KUBERNETES_NAMESPACE_DEV} -f -
                                        
                                        kubectl rollout status deployment/${service} -n ${KUBERNETES_NAMESPACE_DEV} --timeout=300s
                                    """
                                }
                            }
                        }
                    }
                }
                
                stage('Approval for Production') {
                    when {
                        branch 'main'
                    }
                    steps {
                        script {
                            def approver = input(
                                id: 'deploy-approval',
                                message: 'Deploy to production?',
                                submitterParameter: 'APPROVER',
                                parameters: [
                                    choice(
                                        name: 'DEPLOY_DECISION',
                                        choices: ['Deploy', 'Skip'],
                                        description: 'Choose deployment action'
                                    )
                                ]
                            )
                            
                            if (approver.DEPLOY_DECISION != 'Deploy') {
                                error("Deployment cancelled by ${approver.APPROVER}")
                            }
                        }
                    }
                }
                
                stage('Deploy to Production') {
                    when {
                        allOf {
                            branch 'main'
                            expression { env.CHANGED_SERVICES != '' }
                        }
                    }
                    agent {
                        kubernetes {
                            yaml '''
                              apiVersion: v1
                              kind: Pod
                              spec:
                                serviceAccountName: jenkins-deployer-prod
                                containers:
                                - name: kubectl
                                  image: bitnami/kubectl:latest
                                  command: ['cat']
                                  tty: true
                            '''
                        }
                    }
                    steps {
                        container('kubectl') {
                            script {
                                env.CHANGED_SERVICES.split(',').each { service ->
                                    sh """
                                        # Canary deployment
                                        sed 's|{{IMAGE_TAG}}|${BUILD_NUMBER}|g' k8s/${service}.yaml | \\
                                        sed 's|replicas: 3|replicas: 1|g' | \\
                                        kubectl apply -n ${KUBERNETES_NAMESPACE_PROD} -f -
                                        
                                        # Wait and check health
                                        sleep 30
                                        kubectl get pods -n ${KUBERNETES_NAMESPACE_PROD} -l app=${service}
                                        
                                        # Full rollout
                                        sed 's|{{IMAGE_TAG}}|${BUILD_NUMBER}|g' k8s/${service}.yaml | \\
                                        kubectl apply -n ${KUBERNETES_NAMESPACE_PROD} -f -
                                        
                                        kubectl rollout status deployment/${service} -n ${KUBERNETES_NAMESPACE_PROD} --timeout=600s
                                    """
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    
    post {
        success {
            script {
                if (env.BRANCH_NAME == 'main' && env.CHANGED_SERVICES != '') {
                    slackSend(
                        channel: '#deployments',
                        color: 'good',
                        message: """
                            Production deployment successful!
                            
                            Services deployed: ${env.CHANGED_SERVICES}
                            Build: #${BUILD_NUMBER}
                            
                            <${BUILD_URL}|View Build Details>
                        """
                    )
                }
            }
        }
        
        failure {
            slackSend(
                channel: '#ci-cd-alerts',
                color: 'danger',
                message: """
                    Pipeline failed: ${JOB_NAME} #${BUILD_NUMBER}
                    Branch: ${BRANCH_NAME}
                    
                    <${BUILD_URL}console|View Console>
                """
            )
        }
    }
}
```

### 7.3 고급 실습 — 규제 준수 파이프라인

#### 과제 3 — 금융권 수준의 보안 파이프라인

시나리오: PCI DSS와 SOX 규제를 따르는 금융 서비스 배포 파이프라인이다.

요구사항은 다섯 가지다.
1. 모든 단계의 감사 로그 기록
2. 4-eyes 승인 프로세스
3. 보안 스캔 통과 필수
4. 배포 추적성 확보
5. 롤백 자동화

엔터프라이즈 수준 Jenkinsfile
```groovy
pipeline {
    agent none
    
    options {
        // 감사를 위한 빌드 보존
        buildDiscarder(logRotator(
            numToKeepStr: '100',
            daysToKeepStr: '365'
        ))
        
        // 보안을 위한 타임아웃
        timeout(time: 2, unit: 'HOURS')
        
        // 병렬 빌드 방지 (데이터 무결성)
        disableConcurrentBuilds()
    }
    
    environment {
        // 규제 준수를 위한 환경 변수
        COMPLIANCE_SCAN = 'true'
        AUDIT_TRAIL = 'enabled'
        DEPLOYMENT_APPROVAL_REQUIRED = 'true'
        
        // 보안 자격 증명
        SECURITY_SCANNER_TOKEN = credentials('security-scanner-token')
        COMPLIANCE_DB_CREDENTIALS = credentials('compliance-db-credentials')
    }
    
    stages {
        stage('Pre-flight Security Checks') {
            parallel {
                stage('Code Signing Verification') {
                    agent { label 'secure-agent' }
                    steps {
                        sh '''
                            echo "Verifying code signatures..."
                            
                            # GPG 서명 검증
                            git verify-commit HEAD || {
                                echo "Unsigned commit detected"
                                exit 1
                            }
                            
                            # 커밋 작성자 검증
                            AUTHOR_EMAIL=$(git show -s --format='%ae' HEAD)
                            if ! grep -q "$AUTHOR_EMAIL" .approved-contributors; then
                                echo "Unauthorized contributor: $AUTHOR_EMAIL"
                                exit 1
                            fi
                            
                            echo "Code signing verification passed"
                        '''
                    }
                }
                
                stage('Compliance Pre-check') {
                    agent { docker { image 'compliance/scanner:latest' } }
                    steps {
                        sh '''
                            echo "Running compliance pre-checks..."
                            
                            # PCI DSS 요구사항 검증
                            compliance-scanner pci-dss \
                                --config compliance/pci-dss.yaml \
                                --output compliance-report.json
                            
                            # SOX 요구사항 검증
                            compliance-scanner sox \
                                --config compliance/sox.yaml \
                                --append compliance-report.json
                            
                            # 규제 위반 체크
                            VIOLATIONS=$(jq '.violations | length' compliance-report.json)
                            if [ "$VIOLATIONS" -gt 0 ]; then
                                echo "Compliance violations found: $VIOLATIONS"
                                jq '.violations' compliance-report.json
                                exit 1
                            fi
                            
                            echo "Compliance pre-check passed"
                        '''
                    }
                    post {
                        always {
                            // 규제 감사를 위한 리포트 저장
                            archiveArtifacts artifacts: 'compliance-report.json'
                        }
                    }
                }
            }
        }
        
        stage('Comprehensive Security Scanning') {
            parallel {
                stage('SAST - Enterprise Grade') {
                    agent { docker { image 'checkmarx/sast:latest' } }
                    steps {
                        sh '''
                            echo "Running enterprise SAST scan..."
                            
                            checkmarx-cli scan create \
                                --project-name "${JOB_NAME}" \
                                --source-dir . \
                                --incremental false \
                                --preset "High Security" \
                                --report-format json \
                                --output sast-results.json
                            
                            # 높은 심각도 취약점 체크
                            HIGH_VULNS=$(jq '[.results[] | select(.severity == "High")] | length' sast-results.json)
                            CRITICAL_VULNS=$(jq '[.results[] | select(.severity == "Critical")] | length' sast-results.json)
                            
                            echo "Critical vulnerabilities: $CRITICAL_VULNS"
                            echo "High vulnerabilities: $HIGH_VULNS"
                            
                            if [ "$CRITICAL_VULNS" -gt 0 ]; then
                                echo "Critical vulnerabilities found - failing build"
                                exit 1
                            fi
                            
                            if [ "$HIGH_VULNS" -gt 3 ]; then
                                echo "Too many high-severity vulnerabilities"
                                exit 1
                            fi
                        '''
                    }
                }
                
                stage('DAST - Dynamic Analysis') {
                    agent { docker { image 'owasp/zap2docker-stable:latest' } }
                    steps {
                        sh '''
                            echo "Running dynamic application security testing..."
                            
                            # ZAP 베이스라인 스캔
                            zap-baseline.py \
                                -t http://app-staging.company.internal \
                                -J dast-report.json \
                                -r dast-report.html \
                                -x dast-report.xml
                            
                            # 결과 분석
                            python3 << 'EOF'
import json
import sys

with open('dast-report.json', 'r') as f:
    data = json.load(f)

high_alerts = [alert for alert in data.get('site', [{}])[0].get('alerts', []) 
               if alert.get('riskdesc', '').startswith('High')]

if len(high_alerts) > 0:
    print(f"High-risk vulnerabilities found: {len(high_alerts)}")
    for alert in high_alerts:
        print(f"  - {alert.get('name', 'Unknown')}: {alert.get('desc', '')}")
    sys.exit(1)

print("DAST scan passed")
EOF
                        '''
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: '.',
                                reportFiles: 'dast-report.html',
                                reportName: 'DAST Security Report'
                            ])
                        }
                    }
                }
                
                stage('Infrastructure Security') {
                    agent { docker { image 'aquasec/trivy:latest' } }
                    steps {
                        sh '''
                            echo "Scanning infrastructure configurations..."
                            
                            # Docker 이미지 보안 스캔
                            trivy image \
                                --security-checks vuln,config,secret \
                                --format json \
                                --output image-security-report.json \
                                ${DOCKER_REGISTRY}/${APP_NAME}:latest
                            
                            # Kubernetes 매니페스트 스캔
                            trivy config \
                                --format json \
                                --output k8s-security-report.json \
                                k8s/
                            
                            # 보안 정책 위반 체크
                            python3 << 'EOF'
import json
import sys

# 이미지 스캔 결과 분석
with open('image-security-report.json', 'r') as f:
    image_data = json.load(f)

critical_vulns = 0
for result in image_data.get('Results', []):
    for vuln in result.get('Vulnerabilities', []):
        if vuln.get('Severity') == 'CRITICAL':
            critical_vulns += 1

# K8s 스캔 결과 분석
with open('k8s-security-report.json', 'r') as f:
    k8s_data = json.load(f)

k8s_failures = 0
for result in k8s_data.get('Results', []):
    for misconfig in result.get('Misconfigurations', []):
        if misconfig.get('Severity') in ['HIGH', 'CRITICAL']:
            k8s_failures += 1

print(f"Critical image vulnerabilities: {critical_vulns}")
print(f"K8s security misconfigurations: {k8s_failures}")

if critical_vulns > 0 or k8s_failures > 0:
    print("Infrastructure security check failed")
    sys.exit(1)

print("Infrastructure security check passed")
EOF
                        '''
                    }
                }
            }
        }
        
        stage('Security & Compliance Approval') {
            when {
                anyOf {
                    branch 'main'
                    branch 'release/*'
                }
            }
            agent { label 'master' }
            steps {
                script {
                    // 보안팀 승인 프로세스
                    def securityApproval = input(
                        id: 'security-approval',
                        message: 'Security Review Required',
                        submitterParameter: 'SECURITY_APPROVER',
                        parameters: [
                            text(
                                name: 'SECURITY_REVIEW_COMMENTS',
                                description: 'Security review comments',
                                defaultValue: ''
                            ),
                            choice(
                                name: 'SECURITY_DECISION',
                                choices: ['APPROVED', 'REJECTED', 'CONDITIONAL_APPROVED'],
                                description: 'Security approval decision'
                            )
                        ]
                    )
                    
                    if (securityApproval.SECURITY_DECISION == 'REJECTED') {
                        error("Security approval rejected by ${securityApproval.SECURITY_APPROVER}")
                    }
                    
                    // 규제 준수 팀 승인
                    def complianceApproval = input(
                        id: 'compliance-approval',
                        message: 'Compliance Review Required',
                        submitterParameter: 'COMPLIANCE_APPROVER',
                        parameters: [
                            text(
                                name: 'COMPLIANCE_REVIEW_COMMENTS',
                                description: 'Compliance review comments',
                                defaultValue: ''
                            ),
                            choice(
                                name: 'COMPLIANCE_DECISION',
                                choices: ['APPROVED', 'REJECTED'],
                                description: 'Compliance approval decision'
                            )
                        ]
                    )
                    
                    if (complianceApproval.COMPLIANCE_DECISION != 'APPROVED') {
                        error("Compliance approval rejected by ${complianceApproval.COMPLIANCE_APPROVER}")
                    }
                    
                    // 승인 기록을 감사 로그에 저장
                    sh """
                        cat > approval-record.json << 'EOF'
{
    "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ)",
    "build_number": "${BUILD_NUMBER}",
    "git_commit": "${GIT_COMMIT}",
    "security_approval": {
        "approver": "${securityApproval.SECURITY_APPROVER}",
        "decision": "${securityApproval.SECURITY_DECISION}",
        "comments": "${securityApproval.SECURITY_REVIEW_COMMENTS}"
    },
    "compliance_approval": {
        "approver": "${complianceApproval.COMPLIANCE_APPROVER}",
        "decision": "${complianceApproval.COMPLIANCE_DECISION}",
        "comments": "${complianceApproval.COMPLIANCE_REVIEW_COMMENTS}"
    }
}
EOF
                        
                        # 감사 데이터베이스에 기록
                        curl -X POST "https://audit.company.internal/api/approvals" \
                            -H "Content-Type: application/json" \
                            -H "Authorization: Bearer \$AUDIT_API_TOKEN" \
                            -d @approval-record.json
                    """
                }
            }
        }
        
        stage('Controlled Deployment') {
            when {
                allOf {
                    anyOf {
                        branch 'main'
                        branch 'release/*'
                    }
                }
            }
            agent {
                kubernetes {
                    yaml '''
                      apiVersion: v1
                      kind: Pod
                      spec:
                        serviceAccountName: jenkins-deployer-prod
                        containers:
                        - name: kubectl
                          image: bitnami/kubectl:latest
                          command: ['cat']
                          tty: true
                        - name: helm
                          image: alpine/helm:latest
                          command: ['cat']
                          tty: true
                    '''
                }
            }
            stages {
                stage('Blue-Green Deployment') {
                    steps {
                        container('helm') {
                            sh '''
                                echo "Starting blue-green deployment..."
                                
                                # 현재 프로덕션 상태 백업
                                kubectl get deployment ${APP_NAME} -n production -o yaml > current-deployment.yaml
                                
                                # Blue-Green 배포 실행
                                helm upgrade --install ${APP_NAME}-green ./helm-chart \
                                    --namespace production \
                                    --set image.tag=${BUILD_NUMBER} \
                                    --set service.selector.version=green \
                                    --set replicaCount=3 \
                                    --wait --timeout=600s
                                
                                # Health Check
                                echo "Performing health checks..."
                                sleep 30
                                
                                for i in {1..10}; do
                                    STATUS=$(kubectl get pods -n production -l app=${APP_NAME},version=green -o jsonpath='{.items[*].status.phase}')
                                    if echo "$STATUS" | grep -v "Running"; then
                                        echo "Pods not ready, attempt $i/10"
                                        sleep 10
                                    else
                                        echo "All pods are running"
                                        break
                                    fi
                                    
                                    if [ $i -eq 10 ]; then
                                        echo "Health check failed"
                                        exit 1
                                    fi
                                done
                                
                                # 트래픽 전환
                                echo "Switching traffic to green deployment..."
                                kubectl patch service ${APP_NAME} -n production \
                                    -p '{"spec":{"selector":{"version":"green"}}}'
                                
                                # 이전 버전 정리 (5분 후)
                                sleep 300
                                helm uninstall ${APP_NAME}-blue -n production || true
                                
                                echo "Blue-green deployment completed"
                            '''
                        }
                    }
                }
                
                stage('Post-Deployment Verification') {
                    steps {
                        container('kubectl') {
                            sh '''
                                echo "Running post-deployment verification..."
                                
                                # 애플리케이션 health check
                                APP_URL="https://${APP_NAME}.company.com/health"
                                
                                for i in {1..30}; do
                                    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "$APP_URL")
                                    if [ "$HTTP_CODE" = "200" ]; then
                                        echo "Application health check passed"
                                        break
                                    else
                                        echo "Health check attempt $i/30, status: $HTTP_CODE"
                                        sleep 10
                                    fi
                                    
                                    if [ $i -eq 30 ]; then
                                        echo "Application health check failed"
                                        exit 1
                                    fi
                                done
                                
                                # 메트릭 수집 및 알림
                                RESPONSE_TIME=$(curl -s -w "%{time_total}" -o /dev/null "$APP_URL")
                                MEMORY_USAGE=$(kubectl top pod -n production -l app=${APP_NAME} --no-headers | awk '{sum+=$3} END {print sum}')
                                
                                echo "Deployment metrics:"
                                echo "  Response time: ${RESPONSE_TIME}s"
                                echo "  Memory usage: ${MEMORY_USAGE}Mi"
                                
                                # 배포 성공 알림
                                curl -X POST "https://monitoring.company.internal/api/deployments" \
                                    -H "Content-Type: application/json" \
                                    -d '{
                                        "application": "'${APP_NAME}'",
                                        "version": "'${BUILD_NUMBER}'",
                                        "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ)'",
                                        "status": "success",
                                        "metrics": {
                                            "response_time": '${RESPONSE_TIME}',
                                            "memory_usage": '${MEMORY_USAGE}'
                                        }
                                    }'
                            '''
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                // 감사 로그 기록
                sh '''
                    cat > audit-log.json << 'EOF'
{
    "pipeline_execution": {
        "job_name": "'${JOB_NAME}'",
        "build_number": "'${BUILD_NUMBER}'",
        "git_commit": "'${GIT_COMMIT}'",
        "branch": "'${BRANCH_NAME}'",
        "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ)'",
        "duration": "'${currentBuild.duration}'",
        "result": "'${currentBuild.currentResult}'",
        "executor": "'${BUILD_USER_ID}'"
    }
}
EOF
                    
                    # 감사 시스템에 로그 전송
                    curl -X POST "https://audit.company.internal/api/pipeline-logs" \
                        -H "Content-Type: application/json" \
                        -H "Authorization: Bearer $AUDIT_API_TOKEN" \
                        -d @audit-log.json
                '''
                
                // 아티팩트 보관 (규제 요구사항)
                archiveArtifacts artifacts: '**/*-report.json,approval-record.json,audit-log.json', allowEmptyArchive: true
            }
        }
        
        success {
            script {
                if (env.BRANCH_NAME == 'main') {
                    // 성공적인 프로덕션 배포 알림
                    slackSend(
                        channel: '#production-deployments',
                        color: 'good',
                        message: """
                            Production deployment successful!
                            
                            *Application*: ${env.APP_NAME}
                            *Version*: ${env.BUILD_NUMBER}
                            *Commit*: ${env.GIT_COMMIT.take(8)}
                            *Duration*: ${currentBuild.durationString}
                            
                            Security approved by: ${env.SECURITY_APPROVER}
                            Compliance approved by: ${env.COMPLIANCE_APPROVER}
                            
                            <${env.BUILD_URL}|View Build> | <https://${env.APP_NAME}.company.com|View App>
                        """
                    )
                    
                    // 이메일 알림 (경영진)
                    emailext(
                        to: 'executives@company.com, compliance-team@company.com',
                        subject: "Production Deployment Completed: ${env.APP_NAME} v${env.BUILD_NUMBER}",
                        body: """
                            <h2>Production Deployment Notification</h2>
                            
                            <p>A new version of <strong>${env.APP_NAME}</strong> has been successfully deployed to production.</p>
                            
                            <table border="1">
                                <tr><td><strong>Application</strong></td><td>${env.APP_NAME}</td></tr>
                                <tr><td><strong>Version</strong></td><td>${env.BUILD_NUMBER}</td></tr>
                                <tr><td><strong>Git Commit</strong></td><td>${env.GIT_COMMIT}</td></tr>
                                <tr><td><strong>Deployment Time</strong></td><td>${new Date()}</td></tr>
                                <tr><td><strong>Security Approval</strong></td><td>${env.SECURITY_APPROVER}</td></tr>
                                <tr><td><strong>Compliance Approval</strong></td><td>${env.COMPLIANCE_APPROVER}</td></tr>
                            </table>
                            
                            <p>All security scans passed and regulatory requirements have been met.</p>
                            
                            <p><a href="${env.BUILD_URL}">View Deployment Details</a></p>
                        """,
                        mimeType: 'text/html'
                    )
                }
            }
        }
        
        failure {
            script {
                // 실패 시 자동 롤백
                if (env.BRANCH_NAME == 'main' && env.STAGE_NAME?.contains('Deployment')) {
                    sh '''
                        echo "Deployment failed, initiating automatic rollback..."
                        
                        # 이전 안정 버전으로 롤백
                        kubectl rollout undo deployment/${APP_NAME} -n production
                        kubectl rollout status deployment/${APP_NAME} -n production --timeout=300s
                        
                        echo "Automatic rollback completed"
                    '''
                    
                    // 긴급 알림
                    slackSend(
                        channel: '#incident-response',
                        color: 'danger',
                        message: """
                            PRODUCTION DEPLOYMENT FAILED - AUTOMATIC ROLLBACK INITIATED
                            
                            *Application*: ${env.APP_NAME}
                            *Failed Build*: #${env.BUILD_NUMBER}
                            *Branch*: ${env.BRANCH_NAME}
                            *Failed Stage*: ${env.STAGE_NAME}
                            
                            The system has automatically rolled back to the previous stable version.
                            
                            <${env.BUILD_URL}console|View Console Output>
                            
                            @channel @here
                        """
                    )
                    
                    // PagerDuty 알림
                    sh '''
                        curl -X POST https://events.pagerduty.com/v2/enqueue \
                            -H "Content-Type: application/json" \
                            -d '{
                                "routing_key": "'${env.PAGERDUTY_INTEGRATION_KEY}'",
                                "event_action": "trigger",
                                "payload": {
                                    "summary": "Production deployment failure: '${env.APP_NAME}' build #'${env.BUILD_NUMBER}'",
                                    "source": "jenkins-pipeline",
                                    "severity": "critical",
                                    "component": "'${env.APP_NAME}'",
                                    "group": "deployment",
                                    "class": "deployment-failure",
                                    "custom_details": {
                                        "application": "'${env.APP_NAME}'",
                                        "build_number": "'${env.BUILD_NUMBER}'",
                                        "branch": "'${env.BRANCH_NAME}'",
                                        "failed_stage": "'${env.STAGE_NAME}'",
                                        "build_url": "'${env.BUILD_URL}'",
                                        "rollback_status": "completed"
                                    }
                                }
                            }'
                    '''
                }
            }
        }
        
        cleanup {
            // 임시 파일 정리
            sh '''
                rm -f *.json *.xml *.html
                docker system prune -f || true
            '''
            cleanWs()
        }
    }
}
```

## 8. 실전 케이스 스터디

### 8.1 Netflix — 대규모 마이크로서비스 Pipeline

배경: Netflix는 2,500개가 넘는 마이크로서비스를 굴리며, 하루에 4,000회 이상 배포한다.

해결해야 할 문제는 셋이다.
- 서비스 간 의존성 관리
- 배포 실패 시 빠른 롤백
- 카나리 배포로 리스크를 줄이기

Pipeline 전략
```groovy
pipeline {
    agent none
    
    environment {
        SPINNAKER_APPLICATION = env.JOB_NAME.toLowerCase()
        CHAOS_ENGINEERING_ENABLED = 'true'
    }
    
    stages {
        stage('Dependency Analysis') {
            agent { label 'dependency-analyzer' }
            steps {
                sh '''
                    # 서비스 의존성 그래프 생성
                    dependency-analyzer \
                        --service ${SPINNAKER_APPLICATION} \
                        --output dependencies.json
                    
                    # 순환 의존성 검증
                    if dependency-analyzer --check-circular dependencies.json; then
                        echo "No circular dependencies"
                    else
                        echo "Circular dependencies detected"
                        exit 1
                    fi
                '''
            }
        }
        
        stage('Chaos Engineering Tests') {
            when {
                branch 'main'
            }
            agent { docker { image 'netflix/chaosmonkey:latest' } }
            steps {
                sh '''
                    # 카오스 엔지니어링 테스트 실행
                    chaos-monkey \
                        --target-service ${SPINNAKER_APPLICATION} \
                        --failure-modes latency,cpu,memory \
                        --duration 300 \
                        --confidence-threshold 95
                '''
            }
        }
        
        stage('Spinnaker Deployment') {
            agent { docker { image 'spinnaker/spin:latest' } }
            steps {
                sh '''
                    # Spinnaker 파이프라인 트리거
                    spin pipeline execute \
                        --application ${SPINNAKER_APPLICATION} \
                        --name "Deploy to Production" \
                        --parameter imageTag=${BUILD_NUMBER}
                '''
            }
        }
    }
}
```

### 8.2 Google — Monorepo 기반 빌드 최적화

배경: Google은 20억 라인의 코드를 단일 저장소(monorepo)로 관리하며, 빌드 시스템으로 Bazel을 쓴다.

핵심 기술은 셋이다.
- 증분 빌드(Incremental builds)
- 원격 빌드 캐시
- 분산 빌드 실행

Pipeline 구현
```groovy
pipeline {
    agent none
    
    environment {
        BAZEL_CACHE_ENDPOINT = 'https://cache.company.com'
        BUILD_FARM_ENDPOINT = 'grpc://build-farm:8980'
    }
    
    stages {
        stage('Change Detection') {
            agent { label 'change-detector' }
            steps {
                script {
                    // 변경된 타겟만 식별
                    def changedTargets = sh(
                        script: '''
                            bazel query \
                                "rdeps(//..., set($(git diff --name-only HEAD~1 HEAD | grep -E '\\.(java|py|go)$')))" \
                                2>/dev/null | head -100
                        ''',
                        returnStdout: true
                    ).trim().split('\n')
                    
                    env.TARGETS_TO_BUILD = changedTargets.join(' ')
                    echo "Targets to build: ${env.TARGETS_TO_BUILD}"
                }
            }
        }
        
        stage('Distributed Build') {
            when {
                expression { env.TARGETS_TO_BUILD != '' }
            }
            agent {
                kubernetes {
                    yaml '''
                      apiVersion: v1
                      kind: Pod
                      spec:
                        containers:
                        - name: bazel
                          image: gcr.io/bazel-public/bazel:latest
                          resources:
                            requests:
                              memory: "8Gi"
                              cpu: "4"
                            limits:
                              memory: "16Gi"
                              cpu: "8"
                    '''
                }
            }
            steps {
                container('bazel') {
                    sh '''
                        # 분산 빌드 실행
                        bazel build \
                            --remote_executor=${BUILD_FARM_ENDPOINT} \
                            --remote_cache=${BAZEL_CACHE_ENDPOINT} \
                            --jobs=50 \
                            ${TARGETS_TO_BUILD}
                        
                        # 테스트 병렬 실행
                        bazel test \
                            --remote_executor=${BUILD_FARM_ENDPOINT} \
                            --test_output=errors \
                            --flaky_test_attempts=3 \
                            ${TARGETS_TO_BUILD}
                    '''
                }
            }
        }
    }
}
```

### 8.3 Amazon — 보안 중심 Pipeline

배경: Amazon은 보안을 최우선에 두고 파이프라인을 짰다. 그 결과 매분 수백 개의 배포를 안전하게 처리한다.

보안 원칙은 셋이다.
- 최소 권한 원칙
- 모든 단계의 보안 검증
- 자동화된 위협 모델링

Pipeline 구현
```groovy
pipeline {
    agent none
    
    environment {
        AWS_REGION = 'us-west-2'
        SECURITY_HUB_ENABLED = 'true'
        THREAT_MODEL_REQUIRED = 'true'
    }
    
    stages {
        stage('IAM Permission Verification') {
            agent { docker { image 'amazon/aws-cli:latest' } }
            steps {
                sh '''
                    # IAM 권한 최소화 검증
                    aws iam simulate-principal-policy \
                        --policy-source-arn arn:aws:iam::ACCOUNT:role/deployment-role \
                        --action-names s3:GetObject,s3:PutObject,lambda:UpdateFunctionCode \
                        --resource-arns "arn:aws:s3:::deployment-bucket/*","arn:aws:lambda:${AWS_REGION}:ACCOUNT:function:${APP_NAME}"
                '''
            }
        }
        
        stage('Threat Model Analysis') {
            agent { docker { image 'threatmodel/analyzer:latest' } }
            steps {
                sh '''
                    # 자동화된 위협 모델 분석
                    threat-analyzer \
                        --architecture-file threat-model.yaml \
                        --output threat-report.json
                    
                    # 높은 위험도 위협 체크
                    HIGH_RISK_THREATS=$(jq '[.threats[] | select(.risk_level == "HIGH")] | length' threat-report.json)
                    if [ "$HIGH_RISK_THREATS" -gt 0 ]; then
                        echo "High-risk threats identified"
                        jq '.threats[] | select(.risk_level == "HIGH")' threat-report.json
                        exit 1
                    fi
                '''
            }
        }
        
        stage('AWS Security Hub Integration') {
            agent { docker { image 'amazon/aws-cli:latest' } }
            steps {
                sh '''
                    # Security Hub에 파인딩 전송
                    aws securityhub batch-import-findings \
                        --findings file://security-findings.json \
                        --region ${AWS_REGION}
                    
                    # 심각한 파인딩 확인
                    CRITICAL_FINDINGS=$(aws securityhub get-findings \
                        --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"}]}' \
                        --query 'Findings | length' \
                        --output text)
                    
                    if [ "$CRITICAL_FINDINGS" -gt 0 ]; then
                        echo "Critical security findings detected"
                        exit 1
                    fi
                '''
            }
        }
    }
}
```

## 9. 문제 해결 가이드(Troubleshooting)

### 9.1 자주 만나는 파이프라인 오류

#### 9.1.1 Agent 연결 문제

증상
```
ERROR: Couldn't find any revision to build. Verify that the repository is correct and that the branch/tag exists.
```

해결 방법
```groovy
pipeline {
    agent {
        label 'linux && !windows'  // 명시적 라벨 지정
    }
    
    options {
        retry(3)  // 네트워크 오류 대응
        timeout(time: 1, unit: 'HOURS')
    }
    
    stages {
        stage('Debug Agent Info') {
            steps {
                sh '''
                    echo "Node name: ${NODE_NAME}"
                    echo "Workspace: ${WORKSPACE}"
                    echo "Available disk space:"
                    df -h
                    echo "Java version:"
                    java -version
                '''
            }
        }
    }
}
```

#### 9.1.2 Docker 권한 오류

증상
```
permission denied while trying to connect to the Docker daemon socket
```

해결 방법
```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.8.6-openjdk-11'
            args '--group-add docker -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    stages {
        stage('Docker Permission Fix') {
            steps {
                sh '''
                    # Docker 소켓 권한 확인
                    ls -la /var/run/docker.sock
                    
                    # 현재 사용자를 docker 그룹에 추가
                    sudo usermod -aG docker jenkins || true
                    
                    # 그룹 변경사항 적용
                    newgrp docker || true
                '''
            }
        }
    }
}
```

#### 9.1.3 메모리 부족 오류

증상
```
java.lang.OutOfMemoryError: Java heap space
```

해결 방법
```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
              apiVersion: v1
              kind: Pod
              spec:
                containers:
                - name: maven
                  image: maven:3.8.6-openjdk-11
                  env:
                  - name: MAVEN_OPTS
                    value: "-Xmx4g -XX:MaxMetaspaceSize=512m"
                  - name: _JAVA_OPTIONS
                    value: "-Xmx4g"
                  resources:
                    requests:
                      memory: "4Gi"
                      cpu: "2"
                    limits:
                      memory: "8Gi"
                      cpu: "4"
            '''
        }
    }
    
    stages {
        stage('Memory Monitoring') {
            steps {
                sh '''
                    # 메모리 사용량 모니터링
                    free -m
                    
                    # JVM 힙 설정 확인
                    java -XX:+PrintFlagsFinal -version | grep -i heap
                    
                    # 빌드 시작
                    ./mvnw clean compile -X -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn
                '''
            }
        }
    }
}
```

### 9.2 고급 디버깅 기법

#### 9.2.1 파이프라인 실행 시간 분석

```groovy
pipeline {
    agent any
    
    stages {
        stage('Performance Analysis') {
            steps {
                script {
                    def startTime = System.currentTimeMillis()
                    
                    sh '''
                        echo "Starting performance monitoring..."
                        
                        # 시스템 리소스 모니터링
                        top -b -n1 | head -20 > system-stats.txt
                        iostat -x 1 5 > io-stats.txt &
                        
                        # 빌드 실행
                        ./mvnw clean package -B
                        
                        # 모니터링 종료
                        pkill iostat || true
                    '''
                    
                    def endTime = System.currentTimeMillis()
                    def duration = (endTime - startTime) / 1000
                    
                    echo "Stage duration: ${duration} seconds"
                    
                    // 성능 임계값 체크
                    if (duration > 600) { // 10분 초과
                        currentBuild.result = 'UNSTABLE'
                        echo "Stage took longer than expected: ${duration}s"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: '*-stats.txt', allowEmptyArchive: true
                }
            }
        }
    }
}
```

#### 9.2.2 네트워크 연결 진단

```groovy
pipeline {
    agent any
    
    stages {
        stage('Network Diagnostics') {
            steps {
                sh '''
                    echo "=== Network Diagnostics ==="
                    
                    # DNS 해석 테스트
                    nslookup github.com
                    nslookup maven.repository.com
                    
                    # 연결 테스트
                    curl -I -m 10 https://github.com || echo "GitHub unreachable"
                    curl -I -m 10 https://repo1.maven.org/maven2/ || echo "Maven Central unreachable"
                    
                    # 프록시 설정 확인
                    env | grep -i proxy
                    
                    # 방화벽 포트 테스트
                    nc -zv github.com 443
                    nc -zv repo1.maven.org 443
                    
                    echo "=== Network Diagnostics Complete ==="
                '''
            }
        }
    }
}
```

### 9.3 보안 문제 해결

#### 9.3.1 자격 증명 문제

증상
```
Credentials not found for ID: github-token
```

해결 방법
```groovy
pipeline {
    agent any
    
    environment {
        // 자격 증명 ID 검증
        GITHUB_TOKEN = credentials('github-token')
    }
    
    stages {
        stage('Credential Verification') {
            steps {
                sh '''
                    # 자격 증명 존재 여부 확인
                    if [ -z "$GITHUB_TOKEN" ]; then
                        echo "GitHub token not found"
                        echo "Available credentials:"
                        curl -X GET "${JENKINS_URL}/credentials/store/system/domain/_/api/json" \
                            --header "Jenkins-Crumb: $(curl -s "${JENKINS_URL}/crumbIssuer/api/json" | jq -r '.crumb')" \
                            --cookie "${JENKINS_URL}/login"
                        exit 1
                    fi
                    
                    # 토큰 유효성 검증 (마지막 4자리만 표시)
                    echo "GitHub token: ...${GITHUB_TOKEN: -4}"
                    
                    # GitHub API 연결 테스트
                    curl -H "Authorization: token $GITHUB_TOKEN" \
                         -H "Accept: application/vnd.github.v3+json" \
                         https://api.github.com/user | jq '.login'
                '''
            }
        }
    }
}
```

#### 9.3.2 보안 스캔 실패 대응

```groovy
pipeline {
    agent none
    
    stages {
        stage('Security Scan with Fallback') {
            parallel {
                stage('Primary Scanner') {
                    agent { docker { image 'owasp/dependency-check:latest' } }
                    steps {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            sh '''
                                dependency-check.sh \
                                    --project "MyApp" \
                                    --scan . \
                                    --format JSON \
                                    --out dependency-check-report \
                                    --failOnCVSS 7
                            '''
                        }
                    }
                    post {
                        always {
                            script {
                                if (fileExists('dependency-check-report/dependency-check-report.json')) {
                                    def report = readJSON file: 'dependency-check-report/dependency-check-report.json'
                                    def vulnerabilities = report.dependencies?.findAll { 
                                        it.vulnerabilities?.any { vuln -> vuln.cvssv3?.baseScore >= 7.0 } 
                                    }
                                    
                                    if (vulnerabilities) {
                                        echo "High-severity vulnerabilities found:"
                                        vulnerabilities.each { dep ->
                                            echo "  - ${dep.fileName}"
                                        }
                                        
                                        // 취약점 티켓 자동 생성
                                        sh '''
                                            curl -X POST "${JIRA_URL}/rest/api/2/issue" \
                                                -H "Content-Type: application/json" \
                                                -d '{
                                                    "fields": {
                                                        "project": {"key": "SEC"},
                                                        "summary": "Security vulnerabilities found in '${JOB_NAME}' build #'${BUILD_NUMBER}'",
                                                        "description": "High-severity vulnerabilities detected. See build artifacts for details.",
                                                        "issuetype": {"name": "Bug"},
                                                        "priority": {"name": "High"}
                                                    }
                                                }' \
                                                --user "${JIRA_USER}:${JIRA_TOKEN}"
                                        '''
                                    }
                                }
                            }
                        }
                    }
                }
                
                stage('Secondary Scanner') {
                    agent { docker { image 'aquasec/trivy:latest' } }
                    steps {
                        sh '''
                            trivy fs . \
                                --severity HIGH,CRITICAL \
                                --format json \
                                --output trivy-report.json
                            
                            # 결과 분석
                            CRITICAL_COUNT=$(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="CRITICAL")] | length' trivy-report.json)
                            HIGH_COUNT=$(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="HIGH")] | length' trivy-report.json)
                            
                            echo "Critical vulnerabilities: $CRITICAL_COUNT"
                            echo "High vulnerabilities: $HIGH_COUNT"
                            
                            if [ "$CRITICAL_COUNT" -gt 0 ]; then
                                echo "Critical vulnerabilities found"
                                exit 1
                            fi
                        '''
                    }
                }
            }
        }
    }
}
```

## 10. 더 공부하고 싶다면

### 10.1 추가 학습 리소스

공식 문서
- [Jenkins Pipeline Documentation](https://www.jenkins.io/doc/book/pipeline/)
- [Jenkins Plugins Index](https://plugins.jenkins.io/)
- [Blue Ocean User Guide](https://www.jenkins.io/doc/book/blueocean/)

오픈소스 파이프라인 예제
```bash
# Netflix OSS 파이프라인 예제
git clone https://github.com/Netflix/spinnaker-gradle-project
cd spinnaker-gradle-project
cat Jenkinsfile

# Google Kubernetes 파이프라인
git clone https://github.com/GoogleCloudPlatform/microservices-demo
cd microservices-demo
find . -name "Jenkinsfile" -exec cat {} \;

# AWS 파이프라인 베스트 프랙티스
git clone https://github.com/aws-samples/aws-devops-essential
```

실전 연습 환경
```yaml
# docker-compose.yml - 로컬 Jenkins 환경
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JENKINS_OPTS=--httpPort=8080
    
  docker-dind:
    image: docker:20.10-dind
    privileged: true
    ports:
      - "2376:2376"
    environment:
      - DOCKER_TLS_CERTDIR=/certs
    volumes:
      - docker-certs:/certs

volumes:
  jenkins_home:
  docker-certs:
```

### 10.2 인증과 커리어 패스

Jenkins 인증
1. Jenkins Certified Engineer (JCE)
   - 파이프라인 설계 및 구현
   - 플러그인 개발
   - 엔터프라이즈 배포

관련 인증
2. AWS Certified DevOps Engineer
   - AWS에서 CI/CD 구현하기
   - CloudFormation과 파이프라인 통합

3. Google Cloud Professional DevOps Engineer
   - GKE와 Cloud Build 활용
   - SRE 원칙 적용

4. Azure DevOps Engineer Expert
   - Azure DevOps Services
   - GitHub Actions 통합

### 10.3 실무 프로젝트 아이디어

#### 프로젝트 1 — 개인 블로그 자동화 파이프라인
```groovy
// 목표: GitHub Pages 블로그 자동 배포
pipeline {
    agent { docker { image 'ruby:2.7' } }
    
    stages {
        stage('Install Jekyll') {
            steps {
                sh 'gem install bundler jekyll'
            }
        }
        
        stage('Build Site') {
            steps {
                sh 'bundle install'
                sh 'bundle exec jekyll build'
            }
        }
        
        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh 'gh-pages -d _site'
            }
        }
    }
}
```

#### 프로젝트 2 — 마이크로서비스 E2E 테스트 파이프라인
```groovy
// 목표: Docker Compose 기반 통합 테스트
pipeline {
    agent any
    
    stages {
        stage('Start Services') {
            steps {
                sh 'docker-compose up -d'
                sh 'sleep 30'  // 서비스 초기화 대기
            }
        }
        
        stage('E2E Tests') {
            parallel {
                stage('API Tests') {
                    steps {
                        sh 'newman run api-tests/collection.json'
                    }
                }
                stage('UI Tests') {
                    steps {
                        sh 'npm run test:e2e'
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh 'docker-compose down'
        }
    }
}
```

#### 프로젝트 3 — 클라우드 인프라 파이프라인
```groovy
// 목표: Terraform + Ansible 인프라 자동화
pipeline {
    agent { docker { image 'hashicorp/terraform:latest' } }
    
    stages {
        stage('Plan Infrastructure') {
            steps {
                sh '''
                    terraform init
                    terraform plan -out=tfplan
                '''
            }
        }
        
        stage('Apply Infrastructure') {
            when { branch 'main' }
            steps {
                input message: 'Apply infrastructure changes?'
                sh 'terraform apply tfplan'
            }
        }
        
        stage('Configure Servers') {
            steps {
                sh 'ansible-playbook -i inventory.ini playbook.yml'
            }
        }
    }
}
```

## 11. 최종 실습과 평가

### 11.1 종합 실습 과제

시나리오: 당신은 핀테크 스타트업의 DevOps 엔지니어다. 다음 요구사항을 만족하는 파이프라인을 구축해야 한다.

요구사항
1. 보안 스캔 (SAST, DAST, 의존성 검사)
2. 다중 환경 배포 (dev, staging, prod)
3. 승인 프로세스 (staging → prod)
4. 모니터링과 알림
5. 자동 롤백

제공된 애플리케이션
- Spring Boot REST API
- PostgreSQL 데이터베이스
- Redis 캐시
- Kubernetes 배포

평가 기준
- 보안 정책 준수 (40점)
- 파이프라인 효율성 (30점)
- 에러 처리 및 복구 (20점)
- 모니터링 및 알림 (10점)

### 11.2 Self-Assessment Quiz (고급)

Q1. 파이프라인 최적화
파이프라인 실행 시간을 가장 효과적으로 줄이는 방법은?

A) 모든 스테이지를 순차적으로 실행
B) Docker 이미지 크기 최소화
C) 병렬 처리와 빌드 캐시 활용
D) 더 많은 에이전트 노드 추가

정답: C
해설: 동시에 돌릴 수 있는 작업을 병렬로 분산하고 Maven/.gradle, Docker layer 캐시 같은 캐시 자원을 활용하면 실행 시간이 크게 줄어든다.

Q2. 보안 파이프라인 설계
금융권에서 요구하는 보안 파이프라인에서 가장 중요한 원칙은?

A) 빠른 배포 속도
B) 모든 단계에서 보안 검증 수행
C) 사용자 편의성 최우선
D) 비용 절감

정답: B
해설: 금융권은 Shift-Left Security 원칙에 따라 모든 파이프라인 단계에서 보안을 검증한다.

Q3. 파이프라인 실패 대응
프로덕션 배포 도중 실패가 났을 때 올바른 대응 순서는?

A) 로그 분석 → 수정 → 재배포
B) 자동 롤백 → 알림 → 원인 분석 → 수정
C) 서비스 중단 → 수동 복구 → 재시작
D) 사용자 공지 → 개발팀 소집 → 긴급 회의

정답: B
해설: 서비스 안정성을 먼저 확보하도록 자동 롤백을 돌리고, 관련 팀에 즉시 알린 다음 원인을 분석해 수정으로 들어가는 순서가 맞다.

### 11.3 다음 주차 예고

4주차: Advanced Pipeline Techniques
- 조건부 실행과 동적 파이프라인
- Shared Libraries로 코드 재사용
- 파이프라인 모니터링과 메트릭
- 고급 보안 패턴 (Secret Management, RBAC)
- GitOps와 파이프라인 통합

다음 주에는 파이프라인에 지능을 불어넣는다. 상황에 따라 다르게 동작하는 스마트한 파이프라인을 만들고, 엔터프라이즈 환경에서 빼놓을 수 없는 공통 라이브러리 관리 기법을 익힌다.

지금까지 배운 내용만으로도 이미 많은 기업의 CI/CD 파이프라인을 앞선다. Pipeline as Code는 현대 DevOps의 심장이고, 여러분은 이제 인프라를 코드로 다루는 진짜 DevOps 엔지니어다.
