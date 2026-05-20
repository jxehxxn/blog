---
layout: post
title: "DevSecOps Mastery: 4주차 - 고급 파이프라인 기법과 최적화"
date: 2026-05-20 12:45:00 +0900
---

# DevSecOps Mastery 4주차 — 고급 파이프라인 기법과 최적화

## 왜 단순 파이프라인으론 부족한가

지난 3주간 코드형 파이프라인(Pipeline as Code)의 기본기를 다졌다. 그런데 현실의 운영 환경에서는 빌드 → 테스트 → 배포라는 단순 구조만으로는 한계에 부딪힌다.

규모 있는 기업의 사례를 보자. Google은 매일 5만 건 이상의 빌드를 평균 15분 안에 끝내고, Netflix는 하루 4,000회 배포에서 99.9% 성공률을 유지하며, Amazon은 초당 1,000회 넘는 배포가 가능한 인프라를 운영한다.

이만한 규모와 효율을 다루려면 맥락 기반 조건부 실행, 수천 단위 병렬 처리, 환경의 실시간 구성, 그리고 즉각적인 최적화가 필수다.

오늘은 Jenkins 파이프라인을 하루 수천 회 배포 수준으로 끌어올리는 고급 기법들을 직접 만져본다.

## 1. 환경 변수 관리 — 변하는 구성을 다루는 법

### 1.1 환경 변수의 계층 구조와 우선순위

파이프라인의 환경 변수는 여러 층으로 이뤄진다. 이 계층을 정확히 짚어야 결과를 예측할 수 있는 파이프라인을 만든다.

우선순위 (높음 → 낮음):
1. 스크립트 블록 내 직접 정의
2. 스테이지 레벨 environment
3. 파이프라인 레벨 environment
4. Jenkins 글로벌 환경 변수
5. 시스템 환경 변수

```groovy
pipeline {
    agent any
    
    // 파이프라인 레벨 환경 변수
    environment {
        APP_NAME = 'microservice-api'
        BUILD_ENV = 'ci'
        DEFAULT_REGION = 'us-west-2'
        
        // 동적 환경 변수 생성
        BUILD_TIMESTAMP = sh(
            script: 'date +%Y%m%d-%H%M%S',
            returnStdout: true
        ).trim()
        
        // Git 정보 추출
        GIT_SHORT_COMMIT = sh(
            script: 'git rev-parse --short HEAD',
            returnStdout: true
        ).trim()
        
        // 브랜치 기반 환경 결정
        DEPLOY_ENV = script {
            if (env.BRANCH_NAME == 'main') {
                return 'production'
            } else if (env.BRANCH_NAME == 'develop') {
                return 'staging'
            } else {
                return 'development'
            }
        }
    }
    
    stages {
        stage('Environment Setup') {
            environment {
                // 스테이지 레벨 환경 변수 (파이프라인 레벨보다 우선)
                LOG_LEVEL = 'DEBUG'
                TIMEOUT_SECONDS = '300'
            }
            
            steps {
                script {
                    // 스크립트 블록 내 동적 변수 (최고 우선순위)
                    def instanceType = env.DEPLOY_ENV == 'production' ? 't3.large' : 't3.micro'
                    env.INSTANCE_TYPE = instanceType
                    
                    // 환경 정보 출력
                    echo """
                        Environment Configuration:
                        ========================
                        App Name: ${env.APP_NAME}
                        Build Environment: ${env.BUILD_ENV}
                        Deploy Environment: ${env.DEPLOY_ENV}
                        Git Commit: ${env.GIT_SHORT_COMMIT}
                        Build Timestamp: ${env.BUILD_TIMESTAMP}
                        Instance Type: ${env.INSTANCE_TYPE}
                        Log Level: ${env.LOG_LEVEL}
                        Default Region: ${env.DEFAULT_REGION}
                    """
                }
            }
        }
    }
}
```

### 1.2 보안 자격 증명 관리

기본 자격 증명 사용법:
```groovy
pipeline {
    agent any
    
    environment {
        // 단일 문자열 자격 증명
        GITHUB_TOKEN = credentials('github-api-token')
        SONAR_TOKEN = credentials('sonar-analysis-token')
        
        // 사용자명/비밀번호 자격 증명
        DOCKER_REGISTRY = credentials('harbor-registry')
        // 자동으로 DOCKER_REGISTRY_USR, DOCKER_REGISTRY_PSW 변수 생성
        
        // SSH 키 자격 증명
        SSH_KEY = credentials('production-deploy-key')
        
        // 파일 자격 증명
        KUBECONFIG = credentials('kubernetes-config')
        SERVICE_ACCOUNT_KEY = credentials('gcp-service-account')
    }
    
    stages {
        stage('Secure Operations') {
            steps {
                script {
                    // Docker 레지스트리 로그인
                    sh '''
                        echo "$DOCKER_REGISTRY_PSW" | docker login $DOCKER_REGISTRY_USR \
                            --password-stdin harbor.company.com
                    '''
                    
                    // Git 클론 (프라이빗 저장소)
                    sh '''
                        git clone https://$GITHUB_TOKEN@github.com/company/private-repo.git
                    '''
                    
                    // Kubernetes 배포
                    sh '''
                        export KUBECONFIG=$KUBECONFIG
                        kubectl apply -f k8s-manifests/
                    '''
                    
                    // Google Cloud 인증
                    sh '''
                        gcloud auth activate-service-account \
                            --key-file="$SERVICE_ACCOUNT_KEY"
                    '''
                }
            }
        }
    }
}
```

### 1.3 조건부 환경 변수와 매트릭스 구성

다중 환경 매트릭스 관리:
```groovy
pipeline {
    agent none
    
    environment {
        // 기본 환경 변수
        REGISTRY_BASE = 'harbor.company.com'
        NOTIFICATION_CHANNEL = '#ci-cd-alerts'
    }
    
    stages {
        stage('Matrix Build') {
            matrix {
                axes {
                    axis {
                        name 'ENVIRONMENT'
                        values 'development', 'staging', 'production'
                    }
                    axis {
                        name 'PLATFORM'
                        values 'amd64', 'arm64'
                    }
                    axis {
                        name 'RUNTIME'
                        values 'java11', 'java17', 'java21'
                    }
                }
                excludes {
                    exclude {
                        axis {
                            name 'ENVIRONMENT'
                            values 'production'
                        }
                        axis {
                            name 'RUNTIME'
                            values 'java21'
                        }
                    }
                }
                stages {
                    stage('Configure Environment') {
                        environment {
                            // 매트릭스 값 기반 동적 환경 변수
                            IMAGE_TAG = "${RUNTIME}-${PLATFORM}-${BUILD_NUMBER}"
                            REGISTRY_URL = "${REGISTRY_BASE}/${ENVIRONMENT}"
                            
                            // 환경별 리소스 할당
                            MEMORY_LIMIT = script {
                                switch(ENVIRONMENT) {
                                    case 'production':
                                        return '4Gi'
                                    case 'staging':
                                        return '2Gi'
                                    default:
                                        return '1Gi'
                                }
                            }
                            
                            CPU_LIMIT = script {
                                switch(PLATFORM) {
                                    case 'arm64':
                                        return '2'
                                    default:
                                        return '1'
                                }
                            }
                        }
                        
                        agent {
                            kubernetes {
                                yaml """
                                  apiVersion: v1
                                  kind: Pod
                                  spec:
                                    containers:
                                    - name: builder
                                      image: adoptopenjdk:${RUNTIME}-jdk-hotspot
                                      command: ['cat']
                                      tty: true
                                      resources:
                                        requests:
                                          memory: "${MEMORY_LIMIT}"
                                          cpu: "${CPU_LIMIT}"
                                        limits:
                                          memory: "${MEMORY_LIMIT}"
                                          cpu: "${CPU_LIMIT}"
                                """
                            }
                        }
                        
                        steps {
                            container('builder') {
                                script {
                                    echo """
                                        Matrix Configuration:
                                        Environment: ${ENVIRONMENT}
                                        Platform: ${PLATFORM}
                                        Runtime: ${RUNTIME}
                                        Image Tag: ${IMAGE_TAG}
                                        Registry: ${REGISTRY_URL}
                                        Resources: ${CPU_LIMIT} CPU, ${MEMORY_LIMIT} Memory
                                    """
                                    
                                    // 플랫폼별 빌드 명령어
                                    if (PLATFORM == 'arm64') {
                                        sh '''
                                            ./mvnw clean package -B \
                                                -Dquarkus.native.container-build=true \
                                                -Dquarkus.container-image.build=true \
                                                -Dquarkus.container-image.tag=${IMAGE_TAG}
                                        '''
                                    } else {
                                        sh '''
                                            ./mvnw clean package -B \
                                                -Dspring.profiles.active=${ENVIRONMENT}
                                        '''
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
```

### 1.4 외부 시스템 통합 환경 변수

실제 기업 환경 통합 예시:
```groovy
pipeline {
    agent any
    
    environment {
        // 서비스 디스커버리
        CONSUL_URL = 'https://consul.company.internal'
        VAULT_ADDR = 'https://vault.company.internal'
        
        // 모니터링 시스템
        PROMETHEUS_PUSHGATEWAY = 'http://pushgateway:9091'
        GRAFANA_DASHBOARD_URL = 'https://grafana.company.com'
        
        // 알림 시스템
        SLACK_WEBHOOK = credentials('slack-ci-webhook')
        PAGERDUTY_INTEGRATION_KEY = credentials('pagerduty-integration')
        
        // 외부 서비스
        JIRA_URL = 'https://company.atlassian.net'
        CONFLUENCE_URL = 'https://company.atlassian.net/wiki'
        
        // 품질 게이트웨이
        SONARQUBE_URL = 'https://sonar.company.internal'
        NEXUS_URL = 'https://nexus.company.internal'
        
        // 보안 스캐너
        SNYK_API = 'https://api.snyk.io'
        OWASP_DC_URL = 'https://jeremylong.github.io/DependencyCheck'
    }
    
    stages {
        stage('External Integration Test') {
            steps {
                script {
                    // Consul에서 서비스 설정 가져오기
                    def serviceConfig = sh(
                        script: """
                            curl -s "${CONSUL_URL}/v1/kv/services/${APP_NAME}/config" | \
                            jq -r '.[] | .Value' | base64 -d
                        """,
                        returnStdout: true
                    ).trim()
                    
                    // Vault에서 동적 비밀 가져오기
                    def dbCredentials = sh(
                        script: """
                            vault kv get -format=json secret/databases/${ENVIRONMENT} | \
                            jq -r '.data.data | to_entries[] | "\\(.key)=\\(.value)"'
                        """,
                        returnStdout: true
                    ).trim()
                    
                    // 환경에 동적 설정 적용
                    dbCredentials.split('\n').each { credential ->
                        def (key, value) = credential.split('=')
                        env."DB_${key.toUpperCase()}" = value
                    }
                    
                    // 빌드 메트릭 수집
                    sh """
                        # Prometheus Push Gateway로 메트릭 전송
                        echo "jenkins_build_start_time \$(date +%s)" | \
                            curl --data-binary @- \
                            "${PROMETHEUS_PUSHGATEWAY}/metrics/job/jenkins/instance/${JOB_NAME}"
                    """
                    
                    // Slack 알림
                    sh """
                        curl -X POST "${SLACK_WEBHOOK}" \
                            -H "Content-type: application/json" \
                            -d '{
                                "text": "[BUILD] Build started: ${JOB_NAME} #${BUILD_NUMBER}",
                                "channel": "#ci-cd-alerts",
                                "username": "Jenkins",
                                "attachments": [{
                                    "color": "good",
                                    "fields": [{
                                        "title": "Environment",
                                        "value": "${ENVIRONMENT}",
                                        "short": true
                                    }, {
                                        "title": "Commit",
                                        "value": "${GIT_COMMIT.take(8)}",
                                        "short": true
                                    }]
                                }]
                            }'
                    """
                    
                    echo "External integrations configured successfully"
                }
            }
        }
    }
}

## 2. 고급 흐름 제어 — 파이프라인 분기

### 2.1 복합 조건부 실행 (Advanced When Directives)

기본 `branch` 조건을 넘어, 업무 로직을 파이프라인에 직접 새길 수 있다.

고급 When 조건들:
```groovy
pipeline {
    agent any
    
    stages {
        stage('Production Deployment') {
            when {
                allOf {
                    // 모든 조건이 참이어야 함
                    branch 'main'
                    not { changeRequest() }
                    expression { 
                        return currentBuild.currentResult == 'SUCCESS' 
                    }
                    environment name: 'DEPLOY_APPROVED', value: 'true'
                }
            }
            steps {
                echo 'All conditions met, deploying to production...'
            }
        }
        
        stage('Security Scan') {
            when {
                anyOf {
                    // 하나라도 참이면 실행
                    changeset "**/*.java"
                    changeset "**/*.py"
                    changeset "**/Dockerfile"
                    triggeredBy 'TimerTrigger'
                    triggeredBy 'UserIdCause'
                }
            }
            steps {
                echo 'Code changes detected, running security scan...'
            }
        }
        
        stage('Performance Test') {
            when {
                allOf {
                    branch pattern: "release/.*", comparator: "REGEXP"
                    expression {
                        // 마지막 커밋이 24시간 이내인지 확인
                        def lastCommitTime = sh(
                            script: 'git log -1 --format=%ct',
                            returnStdout: true
                        ).trim() as Long
                        def currentTime = System.currentTimeMillis() / 1000
                        return (currentTime - lastCommitTime) < 86400
                    }
                }
            }
            steps {
                echo 'Recent release branch changes, running performance tests...'
            }
        }
        
        stage('Database Migration') {
            when {
                beforeAgent true  // Agent 할당 전에 조건 검사
                anyOf {
                    changeset "**/migrations/**"
                    changeset "**/sql/**"
                    environment name: 'FORCE_MIGRATION', value: 'true'
                }
            }
            agent {
                docker {
                    image 'postgres:13'
                    args '--network=host'
                }
            }
            steps {
                echo 'Database changes detected, running migrations...'
            }
        }
    }
}
```

### 2.2 동적 스테이지 생성

정적으로 굳혀둔 파이프라인이 아니라, 런타임에 스테이지를 만들어 끼우는 패턴이다.

```groovy
pipeline {
    agent any
    
    stages {
        stage('Dynamic Stage Generation') {
            steps {
                script {
                    // Git에서 변경된 서비스 감지
                    def changedServices = sh(
                        script: '''
                            git diff --name-only HEAD~1 HEAD | \
                            grep '^services/' | \
                            cut -d'/' -f2 | \
                            sort | uniq
                        ''',
                        returnStdout: true
                    ).trim().split('\n')
                    
                    // 환경 목록 (실제로는 외부 시스템에서 가져올 수 있음)
                    def environments = ['dev', 'staging', 'prod']
                    def testSuites = ['unit', 'integration', 'e2e']
                    
                    // 각 서비스별 동적 스테이지 생성
                    changedServices.each { service ->
                        if (service && service.trim()) {
                            // 서비스별 테스트 스테이지
                            testSuites.each { testType ->
                                createTestStage(service, testType)
                            }
                            
                            // 서비스별 환경 배포 스테이지
                            environments.each { env ->
                                createDeploymentStage(service, env)
                            }
                        }
                    }
                }
            }
        }
    }
}

def createTestStage(serviceName, testType) {
    return {
        stage("Test: ${serviceName} - ${testType}") {
            when {
                expression {
                    return fileExists("services/${serviceName}/tests/${testType}")
                }
            }
            steps {
                dir("services/${serviceName}") {
                    script {
                        switch(testType) {
                            case 'unit':
                                sh 'npm test'
                                break
                            case 'integration':
                                sh 'npm run test:integration'
                                break
                            case 'e2e':
                                sh 'npm run test:e2e'
                                break
                        }
                    }
                }
            }
            post {
                always {
                    publishTestResults testResultsPattern: "services/${serviceName}/test-results-${testType}.xml"
                }
            }
        }
    }
}

def createDeploymentStage(serviceName, environment) {
    return {
        stage("Deploy: ${serviceName} to ${environment}") {
            when {
                allOf {
                    expression {
                        // 이전 단계들이 성공했는지 확인
                        return currentBuild.currentResult == 'SUCCESS'
                    }
                    expression {
                        // 환경별 배포 조건 확인
                        switch(environment) {
                            case 'dev':
                                return true
                            case 'staging':
                                return env.BRANCH_NAME == 'develop' || env.BRANCH_NAME == 'main'
                            case 'prod':
                                return env.BRANCH_NAME == 'main'
                            default:
                                return false
                        }
                    }
                }
            }
            steps {
                script {
                    echo "Deploying ${serviceName} to ${environment}"
                    
                    // 환경별 배포 로직
                    switch(environment) {
                        case 'dev':
                            sh "kubectl apply -f services/${serviceName}/k8s/dev/ --namespace=dev"
                            break
                        case 'staging':
                            sh "helm upgrade --install ${serviceName} ./helm-charts/${serviceName} --namespace=staging"
                            break
                        case 'prod':
                            // 프로덕션은 Blue-Green 배포
                            sh """
                                helm upgrade --install ${serviceName}-green ./helm-charts/${serviceName} \
                                    --namespace=prod --set color=green
                                sleep 30
                                kubectl patch service ${serviceName} --namespace=prod \
                                    -p '{"spec":{"selector":{"color":"green"}}}'
                                helm uninstall ${serviceName}-blue --namespace=prod || true
                            """
                            break
                    }
                }
            }
        }
    }
}
```

### 2.3 조건부 병렬 실행과 의존성 관리

```groovy
pipeline {
    agent none
    
    stages {
        stage('Conditional Parallel Execution') {
            parallel {
                stage('Frontend Pipeline') {
                    when {
                        changeset "**/frontend/**"
                    }
                    stages {
                        stage('Frontend Build') {
                            agent {
                                docker { image 'node:18' }
                            }
                            steps {
                                dir('frontend') {
                                    sh 'npm ci'
                                    sh 'npm run build'
                                }
                            }
                        }
                        stage('Frontend Test') {
                            agent {
                                docker { image 'node:18' }
                            }
                            steps {
                                dir('frontend') {
                                    sh 'npm run test:unit'
                                    sh 'npm run test:e2e'
                                }
                            }
                        }
                    }
                }
                
                stage('Backend Pipeline') {
                    when {
                        changeset "**/backend/**"
                    }
                    stages {
                        stage('Backend Build') {
                            agent {
                                docker { image 'maven:3.8.6-openjdk-11' }
                            }
                            steps {
                                dir('backend') {
                                    sh 'mvn clean compile'
                                }
                            }
                        }
                        stage('Backend Test') {
                            agent {
                                docker { image 'maven:3.8.6-openjdk-11' }
                            }
                            steps {
                                dir('backend') {
                                    sh 'mvn test'
                                }
                            }
                        }
                    }
                }
                
                stage('Infrastructure Pipeline') {
                    when {
                        changeset "**/terraform/**"
                    }
                    stages {
                        stage('Terraform Plan') {
                            agent {
                                docker { image 'hashicorp/terraform:latest' }
                            }
                            steps {
                                dir('terraform') {
                                    sh 'terraform init'
                                    sh 'terraform plan -out=tfplan'
                                }
                            }
                        }
                        stage('Terraform Apply') {
                            when {
                                allOf {
                                    branch 'main'
                                    environment name: 'TERRAFORM_APPLY_APPROVED', value: 'true'
                                }
                            }
                            agent {
                                docker { image 'hashicorp/terraform:latest' }
                            }
                            steps {
                                dir('terraform') {
                                    sh 'terraform apply tfplan'
                                }
                            }
                        }
                    }
                }
            }
        }
        
        stage('Integration Testing') {
            when {
                expression {
                    // 최소 하나의 서비스가 변경되었을 때만 통합 테스트 실행
                    return currentBuild.changeSets.size() > 0
                }
            }
            agent {
                kubernetes {
                    yaml '''
                      apiVersion: v1
                      kind: Pod
                      spec:
                        containers:
                        - name: integration-test
                          image: postman/newman:latest
                          command: ['cat']
                          tty: true
                    '''
                }
            }
            steps {
                container('integration-test') {
                    script {
                        // 모든 서비스가 준비될 때까지 대기
                        timeout(time: 10, unit: 'MINUTES') {
                            waitUntil {
                                script {
                                    try {
                                        sh '''
                                            curl -f http://frontend:3000/health &&
                                            curl -f http://backend:8080/actuator/health
                                        '''
                                        return true
                                    } catch (Exception e) {
                                        echo "Services not ready yet, waiting..."
                                        sleep 30
                                        return false
                                    }
                                }
                            }
                        }
                        
                        // 통합 테스트 실행
                        sh '''
                            newman run integration-tests/collection.json \
                                --environment integration-tests/environment.json \
                                --reporters junit,html \
                                --reporter-junit-export integration-results.xml \
                                --reporter-html-export integration-results.html
                        '''
                    }
                }
            }
            post {
                always {
                    publishTestResults testResultsPattern: 'integration-results.xml'
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'integration-results.html',
                        reportName: 'Integration Test Report'
                    ])
                }
            }
        }
    }
}
```

### 2.4 타임아웃과 재시도 전략

단일 글로벌 타임아웃에서 그치지 않고, 스테이지·상황별로 세밀하게 제어한다.

```groovy
pipeline {
    agent any
    
    options {
        // 전체 파이프라인 타임아웃
        timeout(time: 2, unit: 'HOURS')
        
        // 재시도 횟수
        retry(3)
        
        // 동시 빌드 방지
        disableConcurrentBuilds()
        
        // 빌드 유지 정책
        buildDiscarder(logRotator(
            numToKeepStr: '10',
            daysToKeepStr: '30'
        ))
    }
    
    stages {
        stage('Fast Tests') {
            options {
                timeout(time: 5, unit: 'MINUTES')
            }
            steps {
                echo 'Running fast tests...'
                sh 'mvn test -Dtest="*UnitTest"'
            }
        }
        
        stage('Integration Tests') {
            options {
                timeout(time: 30, unit: 'MINUTES')
                retry(2)  // 네트워크 문제 등으로 실패할 수 있음
            }
            steps {
                echo 'Running integration tests...'
                sh 'mvn test -Dtest="*IntegrationTest"'
            }
        }
        
        stage('Load Testing') {
            options {
                timeout(time: 1, unit: 'HOURS')
                skipDefaultCheckout()  // 체크아웃 시간 절약
            }
            when {
                anyOf {
                    branch 'main'
                    environment name: 'FORCE_LOAD_TEST', value: 'true'
                }
            }
            steps {
                script {
                    // 커스텀 재시도 로직
                    def maxRetries = 3
                    def currentRetry = 0
                    def success = false
                    
                    while (!success && currentRetry < maxRetries) {
                        try {
                            currentRetry++
                            echo "Load test attempt ${currentRetry}/${maxRetries}"
                            
                            sh '''
                                artillery run load-test-config.yml \
                                    --output load-test-results.json
                            '''
                            
                            // 결과 분석
                            def results = readJSON file: 'load-test-results.json'
                            def avgResponseTime = results.aggregate.latency.mean
                            def errorRate = results.aggregate.errors / results.aggregate.requests
                            
                            if (avgResponseTime > 2000) {
                                throw new Exception("Average response time too high: ${avgResponseTime}ms")
                            }
                            
                            if (errorRate > 0.01) {
                                throw new Exception("Error rate too high: ${errorRate * 100}%")
                            }
                            
                            success = true
                            echo "Load test passed: ${avgResponseTime}ms avg, ${errorRate * 100}% errors"
                            
                        } catch (Exception e) {
                            if (currentRetry >= maxRetries) {
                                error("Load test failed after ${maxRetries} attempts: ${e.getMessage()}")
                            } else {
                                echo "Load test failed, retrying in 60 seconds: ${e.getMessage()}"
                                sleep 60
                            }
                        }
                    }
                }
            }
        }
        
        stage('Deploy with Circuit Breaker') {
            options {
                timeout(time: 15, unit: 'MINUTES')
            }
            steps {
                script {
                    def deploymentSuccess = false
                    def healthCheckAttempts = 0
                    def maxHealthChecks = 10
                    
                    try {
                        // 배포 실행
                        sh '''
                            kubectl set image deployment/myapp \
                                myapp=myregistry/myapp:${BUILD_NUMBER} \
                                --namespace=production
                        '''
                        
                        // 롤아웃 대기
                        timeout(time: 10, unit: 'MINUTES') {
                            sh '''
                                kubectl rollout status deployment/myapp \
                                    --namespace=production \
                                    --timeout=600s
                            '''
                        }
                        
                        // 헬스 체크
                        while (!deploymentSuccess && healthCheckAttempts < maxHealthChecks) {
                            healthCheckAttempts++
                            echo "Health check attempt ${healthCheckAttempts}/${maxHealthChecks}"
                            
                            def healthResponse = sh(
                                script: 'curl -s -o /dev/null -w "%{http_code}" http://myapp.company.com/health',
                                returnStdout: true
                            ).trim()
                            
                            if (healthResponse == '200') {
                                deploymentSuccess = true
                                echo "Deployment health check passed"
                            } else {
                                echo "Health check failed with status ${healthResponse}, retrying in 30 seconds..."
                                sleep 30
                            }
                        }
                        
                        if (!deploymentSuccess) {
                            error("Health checks failed after ${maxHealthChecks} attempts")
                        }
                        
                    } catch (Exception e) {
                        echo "Deployment failed: ${e.getMessage()}"
                        echo "Initiating automatic rollback..."
                        
                        sh '''
                            kubectl rollout undo deployment/myapp \
                                --namespace=production
                            kubectl rollout status deployment/myapp \
                                --namespace=production \
                                --timeout=300s
                        '''
                        
                        error("Deployment failed and rolled back: ${e.getMessage()}")
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo "Pipeline completed with result: ${currentBuild.currentResult}"
        }
        failure {
            script {
                if (env.BRANCH_NAME == 'main') {
                    // 프로덕션 브랜치 실패 시 즉시 알림
                    slackSend(
                        channel: '#alerts',
                        color: 'danger',
                        message: """
                            [CRITICAL] PRODUCTION PIPELINE FAILURE
                            Job: ${env.JOB_NAME}
                            Build: #${env.BUILD_NUMBER}
                            Branch: ${env.BRANCH_NAME}
                            Duration: ${currentBuild.durationString}
                            
                            <${env.BUILD_URL}|View Build>
                        """
                    )
                }
            }
        }
    }
}

## 3. 병렬 실행 — 리소스를 알고 쪼개기

### 3.1 병렬 처리의 아키텍처 원리

단순한 `parallel` 블록만으로는 부족하다. 시스템 리소스와 의존성을 함께 보고 쪼개야 한다.

병렬 처리 최적화 전략:

```groovy
pipeline {
    agent none
    
    environment {
        PARALLEL_JOBS = "${env.NODE_NAME?.contains('master') ? '2' : '8'}"
        MAX_CONCURRENT_BUILDS = '3'
    }
    
    stages {
        stage('Resource-Aware Parallel Processing') {
            parallel {
                stage('CPU-Intensive Tasks') {
                    agent {
                        label 'compute-optimized'
                    }
                    environment {
                        OMP_NUM_THREADS = "${PARALLEL_JOBS}"
                        MAVEN_OPTS = "-Xmx4g -XX:+UseG1GC -XX:ParallelGCThreads=${PARALLEL_JOBS}"
                    }
                    stages {
                        stage('Compilation') {
                            steps {
                                sh """
                                    echo "Using ${PARALLEL_JOBS} parallel jobs for compilation"
                                    mvn clean compile -T ${PARALLEL_JOBS}C
                                """
                            }
                        }
                        stage('Static Analysis') {
                            steps {
                                sh """
                                    # SonarQube 병렬 분석
                                    sonar-scanner -Dsonar.projectKey=myproject \
                                        -Dsonar.java.binaries=target/classes \
                                        -Dsonar.analysis.parallel=true \
                                        -Dsonar.analysis.workers=${PARALLEL_JOBS}
                                """
                            }
                        }
                    }
                }
                
                stage('I/O-Intensive Tasks') {
                    agent {
                        label 'storage-optimized'
                    }
                    stages {
                        stage('Dependency Resolution') {
                            steps {
                                script {
                                    // 의존성 병렬 다운로드
                                    def dependencyTasks = [:]
                                    
                                    ['maven', 'npm', 'docker'].each { tool ->
                                        dependencyTasks["${tool}-deps"] = {
                                            switch(tool) {
                                                case 'maven':
                                                    sh 'mvn dependency:resolve-sources -T 4C'
                                                    break
                                                case 'npm':
                                                    sh 'npm ci --parallel'
                                                    break
                                                case 'docker':
                                                    sh 'docker pull -a myregistry/baseimage'
                                                    break
                                            }
                                        }
                                    }
                                    
                                    parallel dependencyTasks
                                }
                            }
                        }
                        
                        stage('Artifact Upload') {
                            steps {
                                sh """
                                    # 여러 레지스트리에 병렬 업로드
                                    parallel \
                                        "docker push myregistry1/myapp:${BUILD_NUMBER}" \
                                        "docker push myregistry2/myapp:${BUILD_NUMBER}" \
                                        "docker push myregistry3/myapp:${BUILD_NUMBER}"
                                """
                            }
                        }
                    }
                }
                
                stage('Network-Intensive Tasks') {
                    agent {
                        label 'network-optimized'
                    }
                    stages {
                        stage('External API Tests') {
                            steps {
                                script {
                                    // API 엔드포인트별 병렬 테스트
                                    def apiEndpoints = [
                                        'auth': 'https://auth.api.company.com',
                                        'user': 'https://user.api.company.com',
                                        'payment': 'https://payment.api.company.com',
                                        'notification': 'https://notification.api.company.com'
                                    ]
                                    
                                    def apiTasks = [:]
                                    
                                    apiEndpoints.each { name, url ->
                                        apiTasks["api-test-${name}"] = {
                                            retry(3) {
                                                sh """
                                                    newman run api-tests/${name}-collection.json \
                                                        --environment api-tests/environment.json \
                                                        --reporters junit \
                                                        --reporter-junit-export ${name}-api-results.xml
                                                """
                                            }
                                        }
                                    }
                                    
                                    parallel apiTasks
                                }
                            }
                            post {
                                always {
                                    publishTestResults testResultsPattern: '*-api-results.xml'
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

### 3.2 매트릭스 빌드 — 환경 조합 테스트

여러 환경 조합을 한꺼번에 테스트할 때 쓰는 기법이다.

```groovy
pipeline {
    agent none
    
    stages {
        stage('Matrix Testing') {
            matrix {
                axes {
                    axis {
                        name 'JAVA_VERSION'
                        values '8', '11', '17', '21'
                    }
                    axis {
                        name 'OS_TYPE'
                        values 'ubuntu-20.04', 'ubuntu-22.04', 'alpine', 'amazonlinux'
                    }
                    axis {
                        name 'DATABASE'
                        values 'mysql-5.7', 'mysql-8.0', 'postgresql-13', 'postgresql-15'
                    }
                }
                
                excludes {
                    exclude {
                        axis {
                            name 'JAVA_VERSION'
                            values '8'
                        }
                        axis {
                            name 'DATABASE'
                            values 'postgresql-15'
                        }
                    }
                    exclude {
                        axis {
                            name 'OS_TYPE'
                            values 'alpine'
                        }
                        axis {
                            name 'DATABASE'
                            values 'mysql-5.7'
                        }
                    }
                }
                
                stages {
                    stage('Matrix Build and Test') {
                        agent {
                            kubernetes {
                                yaml """
                                  apiVersion: v1
                                  kind: Pod
                                  metadata:
                                    labels:
                                      jenkins: worker
                                  spec:
                                    containers:
                                    - name: java
                                      image: adoptopenjdk:${JAVA_VERSION}-jdk-${OS_TYPE}
                                      command: ['cat']
                                      tty: true
                                      resources:
                                        requests:
                                          memory: "2Gi"
                                          cpu: "1"
                                        limits:
                                          memory: "4Gi"
                                          cpu: "2"
                                    - name: database
                                      image: ${DATABASE.contains('mysql') ? 'mysql' : 'postgres'}:${DATABASE.split('-')[1]}
                                      env:
                                      - name: MYSQL_ROOT_PASSWORD
                                        value: testpassword
                                      - name: POSTGRES_PASSWORD
                                        value: testpassword
                                      - name: POSTGRES_DB
                                        value: testdb
                                """
                            }
                        }
                        
                        environment {
                            TEST_DATABASE_URL = script {
                                if (DATABASE.contains('mysql')) {
                                    return "jdbc:mysql://localhost:3306/test"
                                } else {
                                    return "jdbc:postgresql://localhost:5432/testdb"
                                }
                            }
                            
                            MAVEN_OPTS = script {
                                switch(JAVA_VERSION) {
                                    case '8':
                                        return "-Xmx2g -XX:+UseParallelGC"
                                    case '11':
                                    case '17':
                                        return "-Xmx2g -XX:+UseG1GC"
                                    case '21':
                                        return "-Xmx2g -XX:+UseZGC"
                                    default:
                                        return "-Xmx2g"
                                }
                            }
                        }
                        
                        steps {
                            container('java') {
                                script {
                                    echo """
                                        Matrix Configuration:
                                        - Java Version: ${JAVA_VERSION}
                                        - OS Type: ${OS_TYPE}
                                        - Database: ${DATABASE}
                                        - Database URL: ${TEST_DATABASE_URL}
                                        - Maven Opts: ${MAVEN_OPTS}
                                    """
                                    
                                    // 데이터베이스 준비 대기
                                    timeout(time: 5, unit: 'MINUTES') {
                                        waitUntil {
                                            script {
                                                try {
                                                    if (DATABASE.contains('mysql')) {
                                                        sh 'mysqladmin ping -h localhost -u root --password=testpassword'
                                                    } else {
                                                        sh 'pg_isready -h localhost -U postgres'
                                                    }
                                                    return true
                                                } catch (Exception e) {
                                                    echo "Database not ready, waiting..."
                                                    sleep 10
                                                    return false
                                                }
                                            }
                                        }
                                    }
                                    
                                    // 매트릭스별 특화 테스트 실행
                                    sh """
                                        ./mvnw clean test \
                                            -Dspring.datasource.url=${TEST_DATABASE_URL} \
                                            -Dspring.profiles.active=matrix-test \
                                            -Djava.version=${JAVA_VERSION} \
                                            -Dos.type=${OS_TYPE} \
                                            -Ddatabase.type=${DATABASE}
                                    """
                                    
                                    // 성능 벤치마크 (선택적)
                                    if (JAVA_VERSION == '17' && OS_TYPE == 'ubuntu-22.04') {
                                        sh '''
                                            echo "Running performance benchmark for baseline configuration"
                                            ./mvnw exec:java -Dexec.mainClass="com.company.PerformanceBenchmark" \
                                                -Dexec.args="--iterations=100 --output=benchmark-results.json"
                                        '''
                                        
                                        archiveArtifacts artifacts: 'benchmark-results.json'
                                    }
                                }
                            }
                        }
                        
                        post {
                            always {
                                container('java') {
                                    publishTestResults testResultsPattern: 'target/surefire-reports/*.xml'
                                    
                                    // 매트릭스별 결과 아카이브
                                    archiveArtifacts artifacts: "target/matrix-results-java${JAVA_VERSION}-${OS_TYPE}-${DATABASE}.tar.gz", allowEmptyArchive: true
                                }
                            }
                            failure {
                                script {
                                    // 실패한 매트릭스 조합 추적
                                    writeFile file: "failed-matrix-${JAVA_VERSION}-${OS_TYPE}-${DATABASE}.txt", 
                                              text: "Failed at: ${new Date()}\nBuild: ${BUILD_NUMBER}\nNode: ${NODE_NAME}"
                                    archiveArtifacts artifacts: "failed-matrix-*.txt"
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

### 3.3 동적 병렬 스테이징 및 리소스 최적화

```groovy
pipeline {
    agent none
    
    parameters {
        choice(
            name: 'PARALLELISM_LEVEL',
            choices: ['conservative', 'balanced', 'aggressive'],
            description: 'Parallel execution strategy'
        )
        booleanParam(
            name: 'ENABLE_PERFORMANCE_OPTIMIZATION',
            defaultValue: true,
            description: 'Enable performance optimizations'
        )
    }
    
    stages {
        stage('Dynamic Parallel Resource Allocation') {
            steps {
                script {
                    // 시스템 리소스 분석
                    def availableNodes = nodesByLabel('').size()
                    def systemLoad = sh(
                        script: 'uptime | awk \'{print $(NF-2)}\' | sed \'s/,//\'',
                        returnStdout: true
                    ).trim() as Double
                    
                    // 병렬화 전략 결정
                    def parallelismConfig = [:]
                    
                    switch(params.PARALLELISM_LEVEL) {
                        case 'conservative':
                            parallelismConfig = [
                                maxConcurrentStages: Math.min(2, availableNodes),
                                threadMultiplier: 0.5,
                                memoryPerTask: '2Gi'
                            ]
                            break
                        case 'balanced':
                            parallelismConfig = [
                                maxConcurrentStages: Math.min(4, availableNodes),
                                threadMultiplier: 1.0,
                                memoryPerTask: '1Gi'
                            ]
                            break
                        case 'aggressive':
                            parallelismConfig = [
                                maxConcurrentStages: Math.min(8, availableNodes * 2),
                                threadMultiplier: 1.5,
                                memoryPerTask: '512Mi'
                            ]
                            break
                    }
                    
                    echo """
                        Resource Allocation Configuration:
                        - Available Nodes: ${availableNodes}
                        - System Load: ${systemLoad}
                        - Parallelism Level: ${params.PARALLELISM_LEVEL}
                        - Max Concurrent Stages: ${parallelismConfig.maxConcurrentStages}
                        - Thread Multiplier: ${parallelismConfig.threadMultiplier}
                        - Memory Per Task: ${parallelismConfig.memoryPerTask}
                    """
                    
                    // 동적 병렬 태스크 생성
                    def parallelTasks = [:]
                    def taskConfigurations = [
                        [
                            name: 'Unit Tests',
                            type: 'cpu-intensive',
                            estimatedDuration: 180, // 3분
                            priority: 'high'
                        ],
                        [
                            name: 'Integration Tests',
                            type: 'io-intensive',
                            estimatedDuration: 600, // 10분
                            priority: 'medium'
                        ],
                        [
                            name: 'Security Scan',
                            type: 'cpu-intensive',
                            estimatedDuration: 300, // 5분
                            priority: 'high'
                        ],
                        [
                            name: 'Performance Test',
                            type: 'network-intensive',
                            estimatedDuration: 1200, // 20분
                            priority: 'low'
                        ],
                        [
                            name: 'Code Quality Analysis',
                            type: 'cpu-intensive',
                            estimatedDuration: 240, // 4분
                            priority: 'medium'
                        ]
                    ]
                    
                    // 우선순위와 리소스 타입별 스케줄링
                    taskConfigurations.sort { a, b ->
                        // 높은 우선순위 먼저, 같으면 짧은 작업 먼저
                        def priorityOrder = ['high': 0, 'medium': 1, 'low': 2]
                        def priorityCompare = priorityOrder[a.priority] <=> priorityOrder[b.priority]
                        return priorityCompare != 0 ? priorityCompare : a.estimatedDuration <=> b.estimatedDuration
                    }
                    
                    def currentConcurrentTasks = 0
                    
                    taskConfigurations.each { taskConfig ->
                        if (currentConcurrentTasks < parallelismConfig.maxConcurrentStages) {
                            parallelTasks[taskConfig.name] = createOptimizedTask(
                                taskConfig,
                                parallelismConfig
                            )
                            currentConcurrentTasks++
                        }
                    }
                    
                    // 병렬 실행
                    parallel parallelTasks
                }
            }
        }
    }
}

def createOptimizedTask(taskConfig, parallelismConfig) {
    return {
        stage(taskConfig.name) {
            agent {
                kubernetes {
                    yaml """
                      apiVersion: v1
                      kind: Pod
                      spec:
                        nodeSelector:
                          workload-type: ${taskConfig.type}
                        containers:
                        - name: worker
                          image: ${getOptimizedImage(taskConfig.type)}
                          resources:
                            requests:
                              memory: ${parallelismConfig.memoryPerTask}
                              cpu: "${parallelismConfig.threadMultiplier}"
                            limits:
                              memory: ${parallelismConfig.memoryPerTask}
                              cpu: "${parallelismConfig.threadMultiplier * 2}"
                          env:
                          - name: PARALLEL_THREADS
                            value: "${Math.ceil(parallelismConfig.threadMultiplier * 4)}"
                    """
                }
            }
            
            options {
                timeout(time: taskConfig.estimatedDuration + 60, unit: 'SECONDS')
            }
            
            steps {
                container('worker') {
                    script {
                        def startTime = System.currentTimeMillis()
                        
                        try {
                            switch(taskConfig.name) {
                                case 'Unit Tests':
                                    sh """
                                        mvn test -T \${PARALLEL_THREADS}C \
                                            -Dmaven.test.parallel=methods \
                                            -Dmaven.test.perCoreThreadCount=true
                                    """
                                    break
                                    
                                case 'Integration Tests':
                                    sh """
                                        mvn verify -Dskip.unit.tests=true \
                                            -Dfailsafe.parallel=methods \
                                            -Dfailsafe.threadCount=\${PARALLEL_THREADS}
                                    """
                                    break
                                    
                                case 'Security Scan':
                                    parallel(
                                        'SAST': {
                                            sh 'sonar-scanner -Dsonar.analysis.parallel=true'
                                        },
                                        'Dependency Check': {
                                            sh 'dependency-check.sh --scan . --parallel \${PARALLEL_THREADS}'
                                        },
                                        'Secret Scan': {
                                            sh 'trufflehog --parallel=\${PARALLEL_THREADS} filesystem .'
                                        }
                                    )
                                    break
                                    
                                case 'Performance Test':
                                    sh """
                                        artillery run performance-test.yml \
                                            --config '{"config":{"phases":[{"duration":600,"arrivalRate":10}]}}' \
                                            --parallel \${PARALLEL_THREADS}
                                    """
                                    break
                                    
                                case 'Code Quality Analysis':
                                    parallel(
                                        'PMD': {
                                            sh 'mvn pmd:pmd -Dpmd.threads=\${PARALLEL_THREADS}'
                                        },
                                        'SpotBugs': {
                                            sh 'mvn spotbugs:spotbugs'
                                        },
                                        'Checkstyle': {
                                            sh 'mvn checkstyle:check'
                                        }
                                    )
                                    break
                            }
                            
                            def endTime = System.currentTimeMillis()
                            def actualDuration = (endTime - startTime) / 1000
                            
                            echo """
                                Task Performance:
                                - Task: ${taskConfig.name}
                                - Estimated: ${taskConfig.estimatedDuration}s
                                - Actual: ${actualDuration}s
                                - Efficiency: ${Math.round((taskConfig.estimatedDuration / actualDuration) * 100)}%
                            """
                            
                            // 성능 메트릭 수집
                            sh """
                                curl -X POST "http://prometheus-pushgateway:9091/metrics/job/jenkins-pipeline" \
                                    --data-binary @- << EOF
pipeline_task_duration_seconds{task="${taskConfig.name}",type="${taskConfig.type}"} ${actualDuration}
pipeline_task_efficiency_percent{task="${taskConfig.name}",type="${taskConfig.type}"} ${Math.round((taskConfig.estimatedDuration / actualDuration) * 100)}
EOF
                            """
                            
                        } catch (Exception e) {
                            def endTime = System.currentTimeMillis()
                            def failureDuration = (endTime - startTime) / 1000
                            
                            echo "Task ${taskConfig.name} failed after ${failureDuration}s: ${e.getMessage()}"
                            
                            // 실패 메트릭 기록
                            sh """
                                curl -X POST "http://prometheus-pushgateway:9091/metrics/job/jenkins-pipeline" \
                                    --data-binary @- << EOF
pipeline_task_failure_total{task="${taskConfig.name}",type="${taskConfig.type}"} 1
pipeline_task_failure_duration_seconds{task="${taskConfig.name}",type="${taskConfig.type}"} ${failureDuration}
EOF
                            """
                            
                            throw e
                        }
                    }
                }
            }
        }
    }
}

def getOptimizedImage(workloadType) {
    switch(workloadType) {
        case 'cpu-intensive':
            return 'maven:3.8.6-openjdk-17'
        case 'io-intensive':
            return 'maven:3.8.6-openjdk-11-slim'
        case 'network-intensive':
            return 'postman/newman:latest'
        default:
            return 'maven:3.8.6-openjdk-11'
    }
}

## 4. Shared Libraries — 파이프라인 코드 재사용

### 4.1 Shared Libraries 아키텍처와 구조

조직이 커지면 파이프라인이 수백, 수천 개로 늘어난다. 이때 코드를 한 곳에서 관리하려면 Shared Libraries가 사실상 필수다.

표준 디렉토리 구조:
```
jenkins-shared-library/
├── vars/                    # 글로벌 변수와 파이프라인 함수
│   ├── standardPipeline.groovy
│   ├── deployToK8s.groovy
│   ├── securityScan.groovy
│   ├── notifySlack.groovy
│   └── buildDocker.groovy
├── src/                     # 클래스 라이브러리
│   └── com/
│       └── company/
│           └── jenkins/
│               ├── Pipeline.groovy
│               ├── SecurityUtils.groovy
│               ├── KubernetesDeployer.groovy
│               └── MetricsCollector.groovy
├── resources/               # 정적 리소스
│   ├── templates/
│   │   ├── Dockerfile.template
│   │   ├── k8s-deployment.yaml.template
│   │   └── sonar-project.properties.template
│   └── scripts/
│       ├── security-scan.sh
│       └── performance-test.py
└── test/                    # 테스트
    ├── vars/
    └── src/
```

### 4.2 실무 Shared Library 구현

`vars/standardPipeline.groovy` — 표준 파이프라인 템플릿:
```groovy
def call(Map config) {
    // 기본 설정 병합
    def defaultConfig = [
        javaVersion: '11',
        buildTool: 'maven',
        testFramework: 'junit',
        dockerRegistry: 'harbor.company.com',
        kubernetesNamespace: 'default',
        notificationChannel: '#ci-cd',
        securityScanEnabled: true,
        performanceTestEnabled: false,
        deploymentStrategy: 'rolling-update'
    ]
    
    config = defaultConfig + config
    
    pipeline {
        agent none
        
        options {
            timeout(time: 2, unit: 'HOURS')
            buildDiscarder(logRotator(numToKeepStr: '10'))
            timestamps()
        }
        
        environment {
            APP_NAME = "${config.appName}"
            BUILD_VERSION = "${BUILD_NUMBER}-${GIT_COMMIT.take(8)}"
            REGISTRY = "${config.dockerRegistry}"
            NAMESPACE = "${config.kubernetesNamespace}"
        }
        
        stages {
            stage('Setup') {
                steps {
                    script {
                        // 환경 검증
                        validateEnvironment(config)
                        
                        // 메트릭 수집 시작
                        startMetricsCollection()
                    }
                }
            }
            
            stage('Build') {
                agent {
                    docker {
                        image getBuilderImage(config.buildTool, config.javaVersion)
                        args '-v /var/run/docker.sock:/var/run/docker.sock'
                    }
                }
                steps {
                    script {
                        switch(config.buildTool) {
                            case 'maven':
                                buildWithMaven(config)
                                break
                            case 'gradle':
                                buildWithGradle(config)
                                break
                            case 'npm':
                                buildWithNpm(config)
                                break
                            default:
                                error("Unsupported build tool: ${config.buildTool}")
                        }
                    }
                }
                post {
                    always {
                        archiveArtifacts artifacts: '**/target/*.jar,**/build/libs/*.jar,**/dist/**', allowEmptyArchive: true
                    }
                }
            }
            
            stage('Quality Gates') {
                parallel {
                    stage('Unit Tests') {
                        agent {
                            docker {
                                image getBuilderImage(config.buildTool, config.javaVersion)
                            }
                        }
                        steps {
                            script {
                                runUnitTests(config)
                            }
                        }
                        post {
                            always {
                                publishTestResults testResultsPattern: '**/target/surefire-reports/*.xml,**/build/test-results/**/*.xml,**/test-results.xml'
                                publishHTML([
                                    allowMissing: false,
                                    alwaysLinkToLastBuild: true,
                                    keepAll: true,
                                    reportDir: 'target/site/jacoco,build/reports/jacoco',
                                    reportFiles: 'index.html',
                                    reportName: 'Code Coverage Report'
                                ])
                            }
                        }
                    }
                    
                    stage('Security Scan') {
                        when {
                            expression { return config.securityScanEnabled }
                        }
                        steps {
                            script {
                                securityScan(config)
                            }
                        }
                    }
                    
                    stage('Code Quality') {
                        steps {
                            script {
                                runCodeQualityAnalysis(config)
                            }
                        }
                    }
                }
            }
            
            stage('Docker Build') {
                agent any
                steps {
                    script {
                        buildDocker([
                            appName: config.appName,
                            registry: config.dockerRegistry,
                            buildVersion: env.BUILD_VERSION,
                            dockerfile: config.dockerfile ?: 'Dockerfile'
                        ])
                    }
                }
            }
            
            stage('Deploy') {
                when {
                    anyOf {
                        branch 'main'
                        branch 'develop'
                        environment name: 'FORCE_DEPLOY', value: 'true'
                    }
                }
                steps {
                    script {
                        def targetEnvironment = env.BRANCH_NAME == 'main' ? 'production' : 'staging'
                        
                        deployToK8s([
                            appName: config.appName,
                            namespace: "${targetEnvironment}-${config.kubernetesNamespace}",
                            imageTag: env.BUILD_VERSION,
                            strategy: config.deploymentStrategy,
                            environment: targetEnvironment
                        ])
                    }
                }
            }
            
            stage('Post-Deploy Tests') {
                when {
                    expression { return config.performanceTestEnabled }
                }
                steps {
                    script {
                        runPerformanceTests(config)
                    }
                }
            }
        }
        
        post {
            always {
                script {
                    // 메트릭 수집 종료
                    endMetricsCollection()
                    
                    // 정리 작업
                    cleanupResources()
                }
            }
            success {
                script {
                    notifySlack([
                        channel: config.notificationChannel,
                        color: 'good',
                        message: "[OK] Build successful: ${config.appName} #${BUILD_NUMBER}"
                    ])
                }
            }
            failure {
                script {
                    notifySlack([
                        channel: config.notificationChannel,
                        color: 'danger',
                        message: "[FAIL] Build failed: ${config.appName} #${BUILD_NUMBER}"
                    ])
                    
                    if (env.BRANCH_NAME == 'main') {
                        // 프로덕션 브랜치 실패 시 PagerDuty 알림
                        triggerPagerDuty([
                            severity: 'critical',
                            summary: "Production build failure: ${config.appName}",
                            source: 'jenkins'
                        ])
                    }
                }
            }
        }
    }
}

// Helper 함수들
def validateEnvironment(config) {
    if (!config.appName) {
        error("appName is required")
    }
    
    if (!['maven', 'gradle', 'npm'].contains(config.buildTool)) {
        error("Unsupported build tool: ${config.buildTool}")
    }
    
    echo "Environment validation passed for ${config.appName}"
}

def getBuilderImage(buildTool, javaVersion) {
    switch(buildTool) {
        case 'maven':
            return "maven:3.8.6-openjdk-${javaVersion}"
        case 'gradle':
            return "gradle:7.6-jdk${javaVersion}"
        case 'npm':
            return "node:${javaVersion}-alpine"
        default:
            return "maven:3.8.6-openjdk-11"
    }
}

def buildWithMaven(config) {
    sh """
        mvn clean package -B \
            -DskipTests=${config.skipUnitTests ?: false} \
            -Dmaven.test.failure.ignore=false
    """
}

def buildWithGradle(config) {
    sh """
        ./gradlew clean build \
            ${config.skipUnitTests ? '-x test' : ''} \
            --parallel \
            --build-cache
    """
}

def buildWithNpm(config) {
    sh """
        npm ci
        npm run build
        ${config.skipUnitTests ? '' : 'npm test'}
    """
}

def runUnitTests(config) {
    switch(config.buildTool) {
        case 'maven':
            sh 'mvn test -B'
            break
        case 'gradle':
            sh './gradlew test'
            break
        case 'npm':
            sh 'npm test'
            break
    }
}

def runCodeQualityAnalysis(config) {
    if (config.sonarQubeEnabled) {
        sh """
            sonar-scanner \
                -Dsonar.projectKey=${config.appName} \
                -Dsonar.sources=src/main \
                -Dsonar.tests=src/test \
                -Dsonar.java.binaries=target/classes,build/classes
        """
    }
}

def runPerformanceTests(config) {
    sh """
        artillery run performance-tests/load-test.yml \
            --target ${config.performanceTestTarget} \
            --output performance-results.json
    """
}

def startMetricsCollection() {
    sh """
        curl -X POST http://prometheus-pushgateway:9091/metrics/job/jenkins-build/instance/${JOB_NAME} \
            --data-binary 'jenkins_build_start_time ${System.currentTimeMillis()}'
    """
}

def endMetricsCollection() {
    sh """
        curl -X POST http://prometheus-pushgateway:9091/metrics/job/jenkins-build/instance/${JOB_NAME} \
            --data-binary 'jenkins_build_end_time ${System.currentTimeMillis()}'
        curl -X POST http://prometheus-pushgateway:9091/metrics/job/jenkins-build/instance/${JOB_NAME} \
            --data-binary 'jenkins_build_duration_seconds ${currentBuild.duration / 1000}'
        curl -X POST http://prometheus-pushgateway:9091/metrics/job/jenkins-build/instance/${JOB_NAME} \
            --data-binary 'jenkins_build_result{result="${currentBuild.currentResult}"} 1'
    """
}

def cleanupResources() {
    sh """
        # Docker 정리
        docker system prune -f || true
        
        # 임시 파일 정리
        rm -rf target/temp build/temp || true
        
        # 테스트 데이터베이스 정리
        docker rm -f test-db-${BUILD_NUMBER} || true
    """
}
```

`vars/deployToK8s.groovy` — Kubernetes 배포 함수:
```groovy
def call(Map config) {
    // 필수 파라미터 검증
    def requiredParams = ['appName', 'namespace', 'imageTag']
    requiredParams.each { param ->
        if (!config[param]) {
            error("Required parameter missing: ${param}")
        }
    }
    
    // 기본값 설정
    def deployment = [
        appName: config.appName,
        namespace: config.namespace,
        imageTag: config.imageTag,
        replicas: config.replicas ?: 3,
        strategy: config.strategy ?: 'rolling-update',
        environment: config.environment ?: 'staging',
        resources: config.resources ?: [
            requests: [memory: '512Mi', cpu: '250m'],
            limits: [memory: '1Gi', cpu: '500m']
        ],
        healthCheck: config.healthCheck ?: [
            path: '/health',
            port: 8080,
            initialDelaySeconds: 30,
            periodSeconds: 10
        ]
    ]
    
    echo "Deploying ${deployment.appName}:${deployment.imageTag} to ${deployment.namespace}"
    
    try {
        // 1. Namespace 확인/생성
        sh """
            kubectl get namespace ${deployment.namespace} || \
            kubectl create namespace ${deployment.namespace}
        """
        
        // 2. Secret 및 ConfigMap 적용 (있는 경우)
        if (fileExists("k8s/${deployment.environment}/secrets.yaml")) {
            sh "kubectl apply -f k8s/${deployment.environment}/secrets.yaml -n ${deployment.namespace}"
        }
        
        if (fileExists("k8s/${deployment.environment}/configmap.yaml")) {
            sh "kubectl apply -f k8s/${deployment.environment}/configmap.yaml -n ${deployment.namespace}"
        }
        
        // 3. 배포 전략에 따른 실행
        switch(deployment.strategy) {
            case 'blue-green':
                deployBlueGreen(deployment)
                break
            case 'canary':
                deployCanary(deployment)
                break
            case 'rolling-update':
            default:
                deployRollingUpdate(deployment)
                break
        }
        
        // 4. 배포 후 검증
        verifyDeployment(deployment)
        
        echo "[OK] Deployment successful: ${deployment.appName}:${deployment.imageTag}"
        
    } catch (Exception e) {
        echo "[FAIL] Deployment failed: ${e.getMessage()}"
        
        // 자동 롤백 시도
        if (config.autoRollback != false) {
            rollbackDeployment(deployment)
        }
        
        throw e
    }
}

def deployRollingUpdate(deployment) {
    writeFile file: 'deployment.yaml', text: generateDeploymentManifest(deployment)
    writeFile file: 'service.yaml', text: generateServiceManifest(deployment)
    
    sh """
        kubectl apply -f deployment.yaml -n ${deployment.namespace}
        kubectl apply -f service.yaml -n ${deployment.namespace}
        
        # 롤아웃 대기
        kubectl rollout status deployment/${deployment.appName} \
            -n ${deployment.namespace} \
            --timeout=600s
    """
}

def deployBlueGreen(deployment) {
    def currentColor = getCurrentColor(deployment)
    def nextColor = currentColor == 'blue' ? 'green' : 'blue'
    
    echo "Current deployment color: ${currentColor}, deploying to: ${nextColor}"
    
    // Green 환경에 배포
    def greenDeployment = deployment.clone()
    greenDeployment.color = nextColor
    
    writeFile file: 'deployment-green.yaml', text: generateDeploymentManifest(greenDeployment)
    
    sh """
        kubectl apply -f deployment-green.yaml -n ${deployment.namespace}
        kubectl rollout status deployment/${deployment.appName}-${nextColor} \
            -n ${deployment.namespace} \
            --timeout=600s
    """
    
    // Health check 대기
    waitForHealthCheck(deployment, nextColor)
    
    // 서비스 전환
    sh """
        kubectl patch service ${deployment.appName} -n ${deployment.namespace} \
            -p '{"spec":{"selector":{"color":"${nextColor}"}}}'
    """
    
    echo "Traffic switched to ${nextColor} deployment"
    
    // 구 버전 정리 (5분 후)
    timeout(time: 5, unit: 'MINUTES') {
        sleep 300
        sh """
            kubectl delete deployment ${deployment.appName}-${currentColor} \
                -n ${deployment.namespace} || true
        """
    }
}

def deployCanary(deployment) {
    def canaryReplicas = Math.max(1, Math.ceil(deployment.replicas * 0.1)) // 10%
    def stableReplicas = deployment.replicas - canaryReplicas
    
    echo "Deploying canary: ${canaryReplicas} replicas, stable: ${stableReplicas} replicas"
    
    // 카나리 배포
    def canaryDeployment = deployment.clone()
    canaryDeployment.replicas = canaryReplicas
    canaryDeployment.version = 'canary'
    
    writeFile file: 'deployment-canary.yaml', text: generateDeploymentManifest(canaryDeployment)
    
    sh """
        kubectl apply -f deployment-canary.yaml -n ${deployment.namespace}
        kubectl rollout status deployment/${deployment.appName}-canary \
            -n ${deployment.namespace} \
            --timeout=300s
    """
    
    // 카나리 모니터링 (5분)
    def canarySuccess = monitorCanaryHealth(deployment)
    
    if (canarySuccess) {
        echo "Canary deployment successful, proceeding with full rollout"
        
        // 전체 배포로 전환
        deployment.replicas = deployment.replicas
        writeFile file: 'deployment-full.yaml', text: generateDeploymentManifest(deployment)
        
        sh """
            kubectl apply -f deployment-full.yaml -n ${deployment.namespace}
            kubectl rollout status deployment/${deployment.appName} \
                -n ${deployment.namespace} \
                --timeout=600s
                
            # 카나리 정리
            kubectl delete deployment ${deployment.appName}-canary \
                -n ${deployment.namespace}
        """
    } else {
        error("Canary deployment failed health checks, rolling back")
    }
}

def verifyDeployment(deployment) {
    echo "Verifying deployment health..."
    
    // Pod 상태 확인
    sh """
        kubectl get pods -n ${deployment.namespace} \
            -l app=${deployment.appName} \
            -o jsonpath='{.items[*].status.phase}' | grep -v Running && exit 1 || true
    """
    
    // Health endpoint 확인
    def healthUrl = "http://${deployment.appName}.${deployment.namespace}.svc.cluster.local:${deployment.healthCheck.port}${deployment.healthCheck.path}"
    
    retry(10) {
        sleep 30
        sh """
            kubectl run health-check-${BUILD_NUMBER} \
                --image=curlimages/curl \
                --rm -i --restart=Never \
                --namespace=${deployment.namespace} \
                -- curl -f ${healthUrl}
        """
    }
    
    echo "[OK] Deployment verification completed"
}

def getCurrentColor(deployment) {
    try {
        def color = sh(
            script: """
                kubectl get service ${deployment.appName} \
                    -n ${deployment.namespace} \
                    -o jsonpath='{.spec.selector.color}' 2>/dev/null
            """,
            returnStdout: true
        ).trim()
        return color ?: 'blue'
    } catch (Exception e) {
        return 'blue'
    }
}

def waitForHealthCheck(deployment, color) {
    def maxAttempts = 20
    def attempt = 0
    
    while (attempt < maxAttempts) {
        try {
            sh """
                kubectl run health-check-${color}-${BUILD_NUMBER} \
                    --image=curlimages/curl \
                    --rm -i --restart=Never \
                    --namespace=${deployment.namespace} \
                    -- curl -f http://${deployment.appName}-${color}:${deployment.healthCheck.port}${deployment.healthCheck.path}
            """
            echo "Health check passed for ${color} deployment"
            return
        } catch (Exception e) {
            attempt++
            echo "Health check attempt ${attempt}/${maxAttempts} failed, retrying in 15 seconds..."
            sleep 15
        }
    }
    
    error("Health check failed for ${color} deployment after ${maxAttempts} attempts")
}

def monitorCanaryHealth(deployment) {
    def monitoringDuration = 300 // 5분
    def checkInterval = 30 // 30초
    def checks = monitoringDuration / checkInterval
    def failureThreshold = 0.1 // 10% 에러율
    
    def totalRequests = 0
    def totalErrors = 0
    
    for (int i = 0; i < checks; i++) {
        try {
            // 메트릭 수집
            def metrics = sh(
                script: """
                    kubectl exec -n monitoring deployment/prometheus -- \
                        promtool query instant 'rate(http_requests_total{job="${deployment.appName}-canary"}[1m])'
                """,
                returnStdout: true
            ).trim()
            
            def errorMetrics = sh(
                script: """
                    kubectl exec -n monitoring deployment/prometheus -- \
                        promtool query instant 'rate(http_requests_total{job="${deployment.appName}-canary",status=~"5.."}[1m])'
                """,
                returnStdout: true
            ).trim()
            
            // 간단한 메트릭 파싱 (실제로는 더 정교한 파싱 필요)
            def requestRate = parseMetricValue(metrics)
            def errorRate = parseMetricValue(errorMetrics)
            
            totalRequests += requestRate
            totalErrors += errorRate
            
            def currentErrorRate = totalErrors / Math.max(totalRequests, 1)
            
            echo "Canary monitoring: ${i+1}/${checks}, Error rate: ${currentErrorRate * 100}%"
            
            if (currentErrorRate > failureThreshold) {
                echo "[FAIL] Canary error rate exceeds threshold: ${currentErrorRate * 100}% > ${failureThreshold * 100}%"
                return false
            }
            
            sleep checkInterval
            
        } catch (Exception e) {
            echo "Warning: Failed to collect canary metrics: ${e.getMessage()}"
        }
    }
    
    echo "[OK] Canary monitoring completed successfully"
    return true
}

def parseMetricValue(metricsOutput) {
    // Prometheus 출력 파싱 로직
    // 실제 구현에서는 JSON 파싱 등 더 정교한 처리 필요
    try {
        return metricsOutput.split('\\s+')[-1] as Double
    } catch (Exception e) {
        return 0.0
    }
}

def rollbackDeployment(deployment) {
    echo "[ROLLBACK] Initiating automatic rollback for ${deployment.appName}"
    
    try {
        sh """
            kubectl rollout undo deployment/${deployment.appName} \
                -n ${deployment.namespace}
            kubectl rollout status deployment/${deployment.appName} \
                -n ${deployment.namespace} \
                --timeout=300s
        """
        
        echo "[OK] Rollback completed successfully"
        
    } catch (Exception e) {
        echo "[FAIL] Rollback failed: ${e.getMessage()}"
        
        // 수동 개입 필요 알림
        notifySlack([
            channel: '#incident-response',
            color: 'danger',
            message: """
                [CRITICAL] Automated rollback failed for ${deployment.appName}
                Manual intervention required immediately!
                
                Deployment: ${deployment.appName}:${deployment.imageTag}
                Namespace: ${deployment.namespace}
                Build: ${BUILD_URL}
                
                @channel
            """
        ])
    }
}

def generateDeploymentManifest(deployment) {
    return """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${deployment.appName}${deployment.color ? '-' + deployment.color : ''}${deployment.version ? '-' + deployment.version : ''}
  namespace: ${deployment.namespace}
  labels:
    app: ${deployment.appName}
    version: ${deployment.imageTag}
    ${deployment.color ? 'color: ' + deployment.color : ''}
    ${deployment.version ? 'deployment-type: ' + deployment.version : ''}
spec:
  replicas: ${deployment.replicas}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: ${deployment.appName}
      ${deployment.color ? 'color: ' + deployment.color : ''}
      ${deployment.version ? 'deployment-type: ' + deployment.version : ''}
  template:
    metadata:
      labels:
        app: ${deployment.appName}
        version: ${deployment.imageTag}
        ${deployment.color ? 'color: ' + deployment.color : ''}
        ${deployment.version ? 'deployment-type: ' + deployment.version : ''}
    spec:
      containers:
      - name: ${deployment.appName}
        image: ${deployment.registry ?: 'harbor.company.com'}/${deployment.appName}:${deployment.imageTag}
        ports:
        - containerPort: ${deployment.healthCheck.port}
        resources:
          requests:
            memory: ${deployment.resources.requests.memory}
            cpu: ${deployment.resources.requests.cpu}
          limits:
            memory: ${deployment.resources.limits.memory}
            cpu: ${deployment.resources.limits.cpu}
        livenessProbe:
          httpGet:
            path: ${deployment.healthCheck.path}
            port: ${deployment.healthCheck.port}
          initialDelaySeconds: ${deployment.healthCheck.initialDelaySeconds}
          periodSeconds: ${deployment.healthCheck.periodSeconds}
        readinessProbe:
          httpGet:
            path: ${deployment.healthCheck.path}
            port: ${deployment.healthCheck.port}
          initialDelaySeconds: 10
          periodSeconds: 5
        env:
        - name: ENVIRONMENT
          value: ${deployment.environment}
        - name: BUILD_VERSION
          value: ${deployment.imageTag}
"""
}

def generateServiceManifest(deployment) {
    return """
apiVersion: v1
kind: Service
metadata:
  name: ${deployment.appName}
  namespace: ${deployment.namespace}
  labels:
    app: ${deployment.appName}
spec:
  selector:
    app: ${deployment.appName}
    ${deployment.color ? 'color: ' + deployment.color : ''}
  ports:
  - port: ${deployment.healthCheck.port}
    targetPort: ${deployment.healthCheck.port}
    protocol: TCP
  type: ClusterIP
"""
}
```

### 4.3 Shared Library 사용법

프로젝트의 Jenkinsfile에서 shared library를 불러 쓰는 방법은 이렇다.

```groovy
// Jenkinsfile
@Library('jenkins-shared-library@main') _

standardPipeline([
    appName: 'my-microservice',
    buildTool: 'maven',
    javaVersion: '17',
    dockerRegistry: 'harbor.company.com',
    kubernetesNamespace: 'my-team',
    notificationChannel: '#my-team-alerts',
    securityScanEnabled: true,
    performanceTestEnabled: true,
    deploymentStrategy: 'blue-green',
    resources: [
        requests: [memory: '1Gi', cpu: '500m'],
        limits: [memory: '2Gi', cpu: '1000m']
    ],
    healthCheck: [
        path: '/actuator/health',
        port: 8080,
        initialDelaySeconds: 45
    ]
])
```

커스터마이즈가 필요한 예시:
```groovy
@Library('jenkins-shared-library@main') _

pipeline {
    agent any
    
    stages {
        stage('Custom Pre-Build') {
            steps {
                script {
                    // 커스텀 로직
                    sh 'echo "Custom pre-build logic"'
                }
            }
        }
        
        stage('Standard Build Process') {
            steps {
                script {
                    standardPipeline([
                        appName: 'complex-microservice',
                        buildTool: 'maven',
                        javaVersion: '17',
                        skipUnitTests: false,
                        customBuildCommands: [
                            'mvn clean compile',
                            'mvn spotless:apply',
                            'mvn package -DskipTests=false'
                        ]
                    ])
                }
            }
        }
        
        stage('Custom Integration Tests') {
            parallel {
                stage('Database Integration') {
                    steps {
                        script {
                            // 데이터베이스 통합 테스트
                            sh '''
                                docker run -d --name test-db-${BUILD_NUMBER} \
                                    -e POSTGRES_PASSWORD=test \
                                    -p 5432:5432 postgres:13
                                
                                sleep 30
                                
                                mvn test -Dtest.profile=integration-db
                            '''
                        }
                    }
                    post {
                        always {
                            sh 'docker rm -f test-db-${BUILD_NUMBER} || true'
                        }
                    }
                }
                
                stage('External API Integration') {
                    steps {
                        script {
                            // 외부 API 테스트
                            sh '''
                                newman run integration-tests/external-api.postman_collection.json \
                                    --environment integration-tests/test-environment.json \
                                    --reporters junit \
                                    --reporter-junit-export external-api-results.xml
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Custom Deployment') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // 다중 클러스터 배포
                    def clusters = ['us-west-1', 'us-east-1', 'eu-west-1']
                    def deployTasks = [:]
                    
                    clusters.each { cluster ->
                        deployTasks[cluster] = {
                            deployToK8s([
                                appName: 'complex-microservice',
                                namespace: "production-${cluster}",
                                imageTag: "${BUILD_NUMBER}",
                                strategy: 'canary',
                                cluster: cluster
                            ])
                        }
                    }
                    
                    parallel deployTasks
                }
            }
        }
    }
}
```

## 5. 파이프라인 모니터링과 메트릭 — 가시성 확보

### 5.1 실시간 파이프라인 메트릭 수집

운영 단계에서는 파이프라인의 성능과 안정성을 끊임없이 들여다봐야 한다.

Prometheus + Grafana 통합 예시:
```groovy
pipeline {
    agent any
    
    environment {
        PROMETHEUS_PUSHGATEWAY = 'http://prometheus-pushgateway:9091'
        GRAFANA_DASHBOARD_URL = 'https://grafana.company.com/d/jenkins-pipeline'
    }
    
    stages {
        stage('Metrics Setup') {
            steps {
                script {
                    // 파이프라인 시작 메트릭
                    sendMetric('pipeline_start_time', System.currentTimeMillis())
                    sendMetric('pipeline_start_total', 1)
                    
                    // Git 정보 메트릭
                    def gitCommitCount = sh(
                        script: 'git rev-list --count HEAD',
                        returnStdout: true
                    ).trim() as Integer
                    
                    sendMetric('git_commits_total', gitCommitCount)
                    sendMetric('git_branch_builds_total{branch="' + env.BRANCH_NAME + '"}', 1)
                }
            }
        }
        
        stage('Build with Metrics') {
            steps {
                script {
                    def buildStart = System.currentTimeMillis()
                    
                    try {
                        sh 'mvn clean package -B'
                        
                        def buildDuration = (System.currentTimeMillis() - buildStart) / 1000
                        sendMetric('build_duration_seconds', buildDuration)
                        sendMetric('build_success_total', 1)
                        
                        // 빌드 아티팩트 크기 메트릭
                        def artifactSize = sh(
                            script: 'du -sb target/*.jar | cut -f1',
                            returnStdout: true
                        ).trim() as Long
                        
                        sendMetric('artifact_size_bytes', artifactSize)
                        
                    } catch (Exception e) {
                        sendMetric('build_failure_total', 1)
                        throw e
                    }
                }
            }
        }
        
        stage('Test with Coverage Metrics') {
            steps {
                script {
                    def testStart = System.currentTimeMillis()
                    
                    sh 'mvn test jacoco:report'
                    
                    def testDuration = (System.currentTimeMillis() - testStart) / 1000
                    sendMetric('test_duration_seconds', testDuration)
                    
                    // 테스트 결과 파싱
                    def testResults = parseTestResults()
                    sendMetric('test_total', testResults.total)
                    sendMetric('test_passed_total', testResults.passed)
                    sendMetric('test_failed_total', testResults.failed)
                    sendMetric('test_skipped_total', testResults.skipped)
                    
                    // 코드 커버리지 메트릭
                    def coverage = parseJacocoCoverage()
                    sendMetric('code_coverage_percent', coverage.linecoverage)
                    sendMetric('code_coverage_branch_percent', coverage.branchCoverage)
                }
            }
        }
        
        stage('Performance Metrics Collection') {
            steps {
                script {
                    // 시스템 리소스 사용량
                    def memoryUsage = sh(
                        script: 'free | grep Mem | awk \'{print ($3/$2) * 100.0}\'',
                        returnStdout: true
                    ).trim() as Double
                    
                    def cpuUsage = sh(
                        script: 'top -bn1 | grep "Cpu(s)" | awk \'{print $2 + $4}\'',
                        returnStdout: true
                    ).trim() as Double
                    
                    sendMetric('system_memory_usage_percent', memoryUsage)
                    sendMetric('system_cpu_usage_percent', cpuUsage)
                    
                    // 네트워크 I/O
                    def networkStats = sh(
                        script: 'cat /proc/net/dev | grep eth0 | awk \'{print $2, $10}\'',
                        returnStdout: true
                    ).trim().split(' ')
                    
                    sendMetric('network_bytes_received', networkStats[0] as Long)
                    sendMetric('network_bytes_transmitted', networkStats[1] as Long)
                }
            }
        }
    }
    
    post {
        always {
            script {
                def pipelineDuration = currentBuild.duration / 1000
                sendMetric('pipeline_duration_seconds', pipelineDuration)
                sendMetric('pipeline_end_time', System.currentTimeMillis())
                
                // SLA 메트릭 (10분 기준)
                def slaTarget = 600 // 10분
                def slaCompliant = pipelineDuration <= slaTarget ? 1 : 0
                sendMetric('pipeline_sla_compliant', slaCompliant)
            }
        }
        success {
            script {
                sendMetric('pipeline_success_total', 1)
                calculateSuccessRate()
            }
        }
        failure {
            script {
                sendMetric('pipeline_failure_total', 1)
                analyzeFailurePattern()
            }
        }
    }
}

def sendMetric(metricName, value) {
    def labels = [
        job: env.JOB_NAME,
        branch: env.BRANCH_NAME ?: 'unknown',
        node: env.NODE_NAME ?: 'unknown'
    ].collect { k, v -> "${k}=\"${v}\"" }.join(',')
    
    sh """
        curl -X POST "${PROMETHEUS_PUSHGATEWAY}/metrics/job/jenkins-pipeline/instance/${env.BUILD_NUMBER}" \
            --data-binary '${metricName}{${labels}} ${value}'
    """
}

def parseTestResults() {
    def testReport = readFile('target/surefire-reports/TEST-*.xml')
    // XML 파싱 로직 (실제로는 더 정교한 파싱 필요)
    return [
        total: 150,
        passed: 145,
        failed: 3,
        skipped: 2
    ]
}

def parseJacocoCoverage() {
    if (fileExists('target/site/jacoco/index.html')) {
        def coverage = sh(
            script: '''
                grep -o '[0-9]*%' target/site/jacoco/index.html | head -2 | tr -d '%'
            ''',
            returnStdout: true
        ).trim().split('\n')
        
        return [
            lineCoverage: coverage[0] as Double,
            branchCoverage: coverage[1] as Double
        ]
    }
    return [lineCoverage: 0, branchCoverage: 0]
}

def calculateSuccessRate() {
    // 최근 100개 빌드의 성공률 계산
    def recentBuilds = []
    def currentBuild = currentBuild
    
    for (int i = 0; i < 100 && currentBuild; i++) {
        recentBuilds.add(currentBuild.result == 'SUCCESS')
        currentBuild = currentBuild.previousBuild
    }
    
    def successCount = recentBuilds.count { it }
    def successRate = (successCount / recentBuilds.size()) * 100
    
    sendMetric('pipeline_success_rate_percent', successRate)
}

def analyzeFailurePattern() {
    // 실패 패턴 분석
    def failureStage = currentBuild.rawBuild.getExecution().getFailures()*.displayName.join(',')
    
    sh """
        curl -X POST "${PROMETHEUS_PUSHGATEWAY}/metrics/job/jenkins-pipeline/instance/${env.BUILD_NUMBER}" \
            --data-binary 'pipeline_failure_stage{stage="${failureStage}",job="${env.JOB_NAME}"} 1'
    """
}
```

### 5.2 파이프라인 보안 — 여러 단계 검증

```groovy
pipeline {
    agent none
    
    options {
        // 보안 강화 옵션
        skipDefaultCheckout()
        buildDiscarder(logRotator(
            numToKeepStr: '10',
            artifactNumToKeepStr: '5'
        ))
        timeout(time: 2, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    
    environment {
        // 보안 환경 변수
        SECURITY_SCAN_ENABLED = 'true'
        VULNERABILITY_THRESHOLD = 'high'
        COMPLIANCE_REQUIRED = 'true'
        
        // 보안 도구 설정
        SNYK_TOKEN = credentials('snyk-api-token')
        SONAR_TOKEN = credentials('sonar-security-token')
        TRIVY_CACHE_DIR = '/tmp/trivy-cache'
    }
    
    stages {
        stage('Secure Checkout') {
            agent {
                kubernetes {
                    yaml '''
                      apiVersion: v1
                      kind: Pod
                      spec:
                        securityContext:
                          runAsUser: 1000
                          runAsGroup: 1000
                          fsGroup: 1000
                        containers:
                        - name: git
                          image: alpine/git:latest
                          securityContext:
                            allowPrivilegeEscalation: false
                            readOnlyRootFilesystem: true
                            capabilities:
                              drop:
                              - ALL
                          volumeMounts:
                          - name: workspace
                            mountPath: /workspace
                        volumes:
                        - name: workspace
                          emptyDir: {}
                    '''
                }
            }
            steps {
                container('git') {
                    script {
                        // GPG 서명 검증
                        sh '''
                            git clone ${GIT_URL} /workspace/source
                            cd /workspace/source
                            
                            # 커밋 서명 검증
                            git verify-commit HEAD || {
                                echo "[FAIL] Unsigned commit detected"
                                exit 1
                            }
                            
                            # 신뢰할 수 있는 작성자 확인
                            AUTHOR_EMAIL=$(git show -s --format='%ae' HEAD)
                            if ! grep -q "$AUTHOR_EMAIL" .approved-contributors; then
                                echo "[FAIL] Unauthorized contributor: $AUTHOR_EMAIL"
                                exit 1
                            fi
                            
                            echo "[OK] Secure checkout completed"
                        '''
                    }
                }
            }
        }
        
        stage('Multi-Layer Security Scan') {
            parallel {
                stage('SAST - Static Analysis') {
                    agent {
                        docker {
                            image 'sonarsource/sonar-scanner-cli:latest'
                            args '--user 1000:1000'
                        }
                    }
                    steps {
                        script {
                            sh '''
                                sonar-scanner \
                                    -Dsonar.projectKey=${JOB_NAME} \
                                    -Dsonar.sources=src/main \
                                    -Dsonar.tests=src/test \
                                    -Dsonar.java.binaries=target/classes \
                                    -Dsonar.security.hotspots.threshold=${VULNERABILITY_THRESHOLD} \
                                    -Dsonar.qualitygate.wait=true \
                                    -Dsonar.qualitygate.timeout=300
                            '''
                            
                            // 보안 핫스팟 체크
                            def securityHotspots = sh(
                                script: '''
                                    curl -s "${SONAR_URL}/api/hotspots/search?projectKey=${JOB_NAME}" \
                                        -H "Authorization: Bearer ${SONAR_TOKEN}" | \
                                    jq '.hotspots | length'
                                ''',
                                returnStdout: true
                            ).trim() as Integer
                            
                            if (securityHotspots > 0) {
                                currentBuild.result = 'UNSTABLE'
                                echo "[WARN] ${securityHotspots} security hotspots found"
                            }
                        }
                    }
                }
                
                stage('SCA - Dependency Analysis') {
                    agent {
                        docker {
                            image: 'snyk/snyk:latest'
                            args '--user 1000:1000 -e SNYK_TOKEN'
                        }
                    }
                    steps {
                        sh '''
                            snyk auth ${SNYK_TOKEN}
                            
                            # 의존성 취약점 스캔
                            snyk test \
                                --severity-threshold=${VULNERABILITY_THRESHOLD} \
                                --json > snyk-results.json
                            
                            # 라이선스 스캔
                            snyk test \
                                --license \
                                --json > snyk-license-results.json
                            
                            # Snyk 모니터링에 프로젝트 등록
                            snyk monitor
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'snyk-*.json', allowEmptyArchive: true
                        }
                    }
                }
                
                stage('Secret Detection') {
                    agent {
                        docker {
                            image 'trufflesecurity/trufflehog:latest'
                            args '--user 1000:1000'
                        }
                    }
                    steps {
                        sh '''
                            # 소스 코드 시크릿 스캔
                            trufflehog filesystem . \
                                --json \
                                --no-update \
                                --fail \
                                > secret-scan-results.json
                            
                            # Git 히스토리 시크릿 스캔
                            trufflehog git file://. \
                                --json \
                                --no-update \
                                --fail \
                                > git-secret-scan-results.json
                        '''
                    }
                    post {
                        failure {
                            script {
                                // 시크릿 발견 시 즉시 알림
                                slackSend(
                                    channel: '#security-alerts',
                                    color: 'danger',
                                    message: """
                                        [SECURITY ALERT] Secrets detected in ${env.JOB_NAME}
                                        
                                        Build: #${env.BUILD_NUMBER}
                                        Branch: ${env.BRANCH_NAME}
                                        Commit: ${env.GIT_COMMIT}
                                        
                                        Immediate action required!
                                        <${env.BUILD_URL}|View Details>
                                    """
                                )
                            }
                        }
                    }
                }
                
                stage('Container Security') {
                    agent {
                        docker {
                            image 'aquasec/trivy:latest'
                            args '--user 1000:1000'
                        }
                    }
                    steps {
                        script {
                            // Docker 이미지 빌드
                            sh '''
                                docker build -t temp-security-scan:${BUILD_NUMBER} .
                            '''
                            
                            // 컨테이너 이미지 보안 스캔
                            sh '''
                                trivy image \
                                    --security-checks vuln,config,secret \
                                    --severity HIGH,CRITICAL \
                                    --format json \
                                    --output container-scan-results.json \
                                    temp-security-scan:${BUILD_NUMBER}
                                
                                # 취약점 개수 확인
                                CRITICAL_VULNS=$(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="CRITICAL")] | length' container-scan-results.json)
                                HIGH_VULNS=$(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="HIGH")] | length' container-scan-results.json)
                                
                                echo "Critical vulnerabilities: $CRITICAL_VULNS"
                                echo "High vulnerabilities: $HIGH_VULNS"
                                
                                # 임계값 체크
                                if [ "$CRITICAL_VULNS" -gt 0 ]; then
                                    echo "[FAIL] Critical vulnerabilities found - blocking deployment"
                                    exit 1
                                elif [ "$HIGH_VULNS" -gt 10 ]; then
                                    echo "[WARN] Too many high-severity vulnerabilities"
                                    exit 1
                                fi
                            '''
                        }
                    }
                    post {
                        always {
                            sh 'docker rmi temp-security-scan:${BUILD_NUMBER} || true'
                        }
                    }
                }
                
                stage('Infrastructure Security') {
                    agent {
                        docker {
                            image 'bridgecrew/checkov:latest'
                            args '--user 1000:1000'
                        }
                    }
                    steps {
                        sh '''
                            # Infrastructure as Code 보안 스캔
                            checkov -d . \
                                --framework dockerfile,kubernetes,terraform \
                                --soft-fail \
                                --output json \
                                --output-file iac-security-results.json
                            
                            # 보안 정책 위반 체크
                            FAILED_CHECKS=$(jq '.summary.failed' iac-security-results.json)
                            
                            if [ "$FAILED_CHECKS" -gt 5 ]; then
                                echo "[FAIL] Too many infrastructure security failures: $FAILED_CHECKS"
                                exit 1
                            fi
                        '''
                    }
                }
            }
        }
        
        stage('Compliance Validation') {
            when {
                environment name: 'COMPLIANCE_REQUIRED', value: 'true'
            }
            agent {
                docker {
                    image 'compliance/validator:latest'
                    args '--user 1000:1000'
                }
            }
            steps {
                script {
                    // SOC 2, PCI DSS, GDPR 컴플라이언스 체크
                    sh '''
                        # SOC 2 Type II 체크리스트
                        compliance-checker soc2 \
                            --config compliance/soc2-controls.yaml \
                            --evidence-dir evidence/ \
                            --output soc2-compliance-report.json
                        
                        # PCI DSS 요구사항 체크
                        compliance-checker pci-dss \
                            --config compliance/pci-dss-requirements.yaml \
                            --output pci-dss-compliance-report.json
                        
                        # GDPR 개인정보 처리 체크
                        compliance-checker gdpr \
                            --config compliance/gdpr-controls.yaml \
                            --output gdpr-compliance-report.json
                    '''
                    
                    // 컴플라이언스 결과 분석
                    def complianceResults = readJSON file: 'soc2-compliance-report.json'
                    def violations = complianceResults.violations?.size() ?: 0
                    
                    if (violations > 0) {
                        currentBuild.result = 'UNSTABLE'
                        echo "[WARN] ${violations} compliance violations found"
                        
                        // 컴플라이언스 팀에 알림
                        emailext(
                            to: 'compliance-team@company.com',
                            subject: "Compliance Violations: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                            body: """
                                Compliance violations detected in ${env.JOB_NAME} build #${env.BUILD_NUMBER}
                                
                                Violations: ${violations}
                                Branch: ${env.BRANCH_NAME}
                                Commit: ${env.GIT_COMMIT}
                                
                                Please review the compliance report for details.
                                
                                Build URL: ${env.BUILD_URL}
                            """
                        )
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: '*-compliance-report.json', allowEmptyArchive: true
                }
            }
        }
        
        stage('Security Approval Gate') {
            when {
                allOf {
                    branch 'main'
                    environment name: 'SECURITY_SCAN_ENABLED', value: 'true'
                }
            }
            steps {
                script {
                    // 자동 보안 승인 로직
                    def securityScore = calculateSecurityScore()
                    echo "Security Score: ${securityScore}/100"
                    
                    if (securityScore >= 80) {
                        echo "[OK] Automatic security approval (score: ${securityScore})"
                    } else if (securityScore >= 60) {
                        echo "[WARN] Manual security review required (score: ${securityScore})"
                        
                        // 보안팀 수동 승인 요청
                        def approval = input(
                            id: 'security-approval',
                            message: 'Security Review Required',
                            submitterParameter: 'APPROVER',
                            parameters: [
                                text(
                                    name: 'SECURITY_REVIEW_NOTES',
                                    description: 'Security review notes',
                                    defaultValue: ''
                                ),
                                choice(
                                    name: 'SECURITY_DECISION',
                                    choices: ['APPROVED', 'CONDITIONAL_APPROVED', 'REJECTED'],
                                    description: 'Security approval decision'
                                )
                            ]
                        )
                        
                        if (approval.SECURITY_DECISION == 'REJECTED') {
                            error("Security approval rejected by ${approval.APPROVER}")
                        }
                        
                        // 승인 로그 기록
                        writeFile file: 'security-approval.log', text: """
                            Security Approval Log
                            ==================
                            Timestamp: ${new Date()}
                            Build: ${env.BUILD_NUMBER}
                            Commit: ${env.GIT_COMMIT}
                            Approver: ${approval.APPROVER}
                            Decision: ${approval.SECURITY_DECISION}
                            Notes: ${approval.SECURITY_REVIEW_NOTES}
                            Security Score: ${securityScore}
                        """
                        
                        archiveArtifacts artifacts: 'security-approval.log'
                        
                    } else {
                        error("[FAIL] Security score too low for deployment: ${securityScore}/100")
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                // 보안 메트릭 수집
                collectSecurityMetrics()
                
                // 보안 리포트 생성
                generateSecurityReport()
            }
        }
        failure {
            script {
                // 보안 실패 시 인시던트 생성
                if (currentBuild.rawBuild.getLog(100).any { it.contains('security') || it.contains('vulnerability') }) {
                    createSecurityIncident()
                }
            }
        }
    }
}

def calculateSecurityScore() {
    def score = 100
    
    // 취약점에 따른 점수 차감
    try {
        def criticalVulns = sh(
            script: 'jq "[.Results[]?.Vulnerabilities[]? | select(.Severity==\\"CRITICAL\\")] | length" container-scan-results.json',
            returnStdout: true
        ).trim() as Integer
        
        def highVulns = sh(
            script: 'jq "[.Results[]?.Vulnerabilities[]? | select(.Severity==\\"HIGH\\")] | length" container-scan-results.json',
            returnStdout: true
        ).trim() as Integer
        
        score -= (criticalVulns * 20) + (highVulns * 5)
        
    } catch (Exception e) {
        echo "Warning: Could not parse vulnerability results"
    }
    
    // 컴플라이언스 위반에 따른 점수 차감
    try {
        def violations = readJSON file: 'soc2-compliance-report.json'
        score -= (violations.violations?.size() ?: 0) * 10
    } catch (Exception e) {
        echo "Warning: Could not parse compliance results"
    }
    
    return Math.max(0, score)
}

def collectSecurityMetrics() {
    // 보안 메트릭을 Prometheus로 전송
    sh '''
        curl -X POST "http://prometheus-pushgateway:9091/metrics/job/jenkins-security" \
            --data-binary @- << EOF
security_scan_duration_seconds{job="${JOB_NAME}"} ${BUILD_DURATION}
security_vulnerabilities_critical{job="${JOB_NAME}"} $(jq "[.Results[]?.Vulnerabilities[]? | select(.Severity==\\"CRITICAL\\")] | length" container-scan-results.json 2>/dev/null || echo 0)
security_vulnerabilities_high{job="${JOB_NAME}"} $(jq "[.Results[]?.Vulnerabilities[]? | select(.Severity==\\"HIGH\\")] | length" container-scan-results.json 2>/dev/null || echo 0)
security_compliance_violations{job="${JOB_NAME}"} $(jq ".violations | length" soc2-compliance-report.json 2>/dev/null || echo 0)
EOF
    '''
}

def generateSecurityReport() {
    def report = """
# Security Scan Report
## Build: ${env.BUILD_NUMBER}
## Date: ${new Date()}

### Vulnerability Summary
- Critical: $(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="CRITICAL")] | length' container-scan-results.json 2>/dev/null || echo "N/A")
- High: $(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="HIGH")] | length' container-scan-results.json 2>/dev/null || echo "N/A")

### Compliance Status
- SOC 2: $([ -f soc2-compliance-report.json ] && echo "Scanned" || echo "Not Scanned")
- PCI DSS: $([ -f pci-dss-compliance-report.json ] && echo "Scanned" || echo "Not Scanned")

### Security Score
$(calculateSecurityScore())/100
"""
    
    writeFile file: 'security-report.md', text: report
    archiveArtifacts artifacts: 'security-report.md'
}

def createSecurityIncident() {
    sh '''
        curl -X POST "https://incident-management.company.com/api/incidents" \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer ${INCIDENT_API_TOKEN}" \
            -d '{
                "title": "Security Pipeline Failure: '"${JOB_NAME}"' #'"${BUILD_NUMBER}"'",
                "description": "Security scan or compliance check failed in Jenkins pipeline",
                "severity": "high",
                "tags": ["security", "jenkins", "pipeline"],
                "metadata": {
                    "job_name": "'"${JOB_NAME}"'",
                    "build_number": "'"${BUILD_NUMBER}"'",
                    "branch": "'"${BRANCH_NAME}"'",
                    "commit": "'"${GIT_COMMIT}"'"
                }
            }'
    '''
}
```

## 6. 실전 케이스 — 대기업 파이프라인 아키텍처

### 6.1 Netflix — 마이크로서비스 최적화 사례

배경: Netflix는 전 세계 2억 명이 넘는 사용자를 상대하며, 하루 4,000회 이상 배포한다.

핵심 최적화 전략:
```groovy
// Netflix-style 마이크로서비스 파이프라인
@Library('spinnaker-pipeline-library@main') _

pipeline {
    agent none
    
    environment {
        SPINNAKER_APPLICATION = env.JOB_NAME.toLowerCase()
        CANARY_ANALYSIS_ENABLED = 'true'
        CHAOS_ENGINEERING_ENABLED = 'true'
        AUTOMATIC_ROLLBACK_ENABLED = 'true'
    }
    
    stages {
        stage('Pre-flight Checks') {
            parallel {
                stage('Service Dependencies') {
                    agent { label 'dependency-analyzer' }
                    steps {
                        script {
                            // 서비스 의존성 그래프 분석
                            def dependencies = analyzeDependencies(env.SPINNAKER_APPLICATION)
                            
                            // 순환 의존성 검출
                            if (dependencies.circularDependencies) {
                                error("[FAIL] Circular dependencies detected: ${dependencies.circularDependencies}")
                            }
                            
                            // 의존 서비스 가용성 확인
                            dependencies.services.each { service ->
                                def healthCheck = sh(
                                    script: "curl -f http://${service}.internal/health",
                                    returnStatus: true
                                )
                                if (healthCheck != 0) {
                                    echo "[WARN] Dependency ${service} is unhealthy"
                                }
                            }
                        }
                    }
                }
                
                stage('Performance Baseline') {
                    agent { label 'performance-analyzer' }
                    steps {
                        script {
                            // 현재 프로덕션 성능 기준선 수집
                            def baseline = collectPerformanceBaseline(env.SPINNAKER_APPLICATION)
                            env.PERFORMANCE_BASELINE = writeJSON returnText: true, json: baseline
                            
                            echo "Performance Baseline: ${baseline.avgResponseTime}ms, ${baseline.throughput} RPS"
                        }
                    }
                }
                
                stage('Capacity Planning') {
                    agent { label 'capacity-planner' }
                    steps {
                        script {
                            // 트래픽 예측 및 용량 계획
                            def capacityPlan = planCapacity(
                                application: env.SPINNAKER_APPLICATION,
                                timeHorizon: '24h',
                                scalingFactor: 1.2
                            )
                            
                            echo "Estimated peak capacity needed: ${capacityPlan.peakInstances} instances"
                        }
                    }
                }
            }
        }
        
        stage('Build & Validate') {
            parallel {
                stage('Application Build') {
                    agent {
                        kubernetes {
                            yaml """
                              apiVersion: v1
                              kind: Pod
                              spec:
                                containers:
                                - name: gradle
                                  image: gradle:7.6-jdk11
                                  resources:
                                    requests:
                                      memory: "4Gi"
                                      cpu: "2"
                                    limits:
                                      memory: "8Gi"
                                      cpu: "4"
                            """
                        }
                    }
                    steps {
                        container('gradle') {
                            sh '''
                                ./gradlew clean build \
                                    --parallel \
                                    --build-cache \
                                    --configuration-cache
                            '''
                        }
                    }
                }
                
                stage('Contract Testing') {
                    agent { docker { image 'pactfoundation/pact-cli:latest' } }
                    steps {
                        script {
                            // Pact를 이용한 컨트랙트 테스트
                            sh '''
                                # Consumer 계약 테스트
                                pact-broker can-i-deploy \
                                    --pacticipant ${SPINNAKER_APPLICATION} \
                                    --version ${BUILD_NUMBER} \
                                    --to production
                                
                                # Provider 검증 테스트
                                pact-verifier verify \
                                    --pact-broker-base-url https://pact-broker.internal \
                                    --provider ${SPINNAKER_APPLICATION}
                            '''
                        }
                    }
                }
                
                stage('Chaos Engineering Prep') {
                    when {
                        environment name: 'CHAOS_ENGINEERING_ENABLED', value: 'true'
                    }
                    agent { docker { image 'netflix/chaosmonkey:latest' } }
                    steps {
                        script {
                            // Chaos Engineering 시나리오 준비
                            sh '''
                                chaos-toolkit validate chaos-experiments/
                                chaos-toolkit discover \
                                    --service ${SPINNAKER_APPLICATION} \
                                    --environment staging \
                                    > chaos-discovery.json
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Deployment to Staging') {
            steps {
                script {
                    // Spinnaker 파이프라인 트리거
                    def spinnakerPipeline = triggerSpinnakerPipeline([
                        application: env.SPINNAKER_APPLICATION,
                        pipeline: 'Deploy to Staging',
                        parameters: [
                            buildNumber: env.BUILD_NUMBER,
                            gitCommit: env.GIT_COMMIT,
                            branch: env.BRANCH_NAME
                        ]
                    ])
                    
                    // 배포 완료까지 대기
                    waitForSpinnakerExecution(spinnakerPipeline.executionId)
                }
            }
        }
        
        stage('Staging Validation') {
            parallel {
                stage('Functional Testing') {
                    steps {
                        script {
                            runFunctionalTests(
                                environment: 'staging',
                                testSuite: 'regression'
                            )
                        }
                    }
                }
                
                stage('Performance Testing') {
                    steps {
                        script {
                            def performanceResults = runPerformanceTests(
                                environment: 'staging',
                                duration: '10m',
                                rampUp: '2m'
                            )
                            
                            // 성능 기준선과 비교
                            def baseline = readJSON text: env.PERFORMANCE_BASELINE
                            def regression = performanceResults.avgResponseTime > baseline.avgResponseTime * 1.1
                            
                            if (regression) {
                                currentBuild.result = 'UNSTABLE'
                                echo "[WARN] Performance regression detected"
                            }
                        }
                    }
                }
                
                stage('Chaos Experiments') {
                    when {
                        environment name: 'CHAOS_ENGINEERING_ENABLED', value: 'true'
                    }
                    steps {
                        script {
                            // Controlled chaos experiments
                            sh '''
                                chaos-toolkit run chaos-experiments/latency-injection.yaml \
                                    --var service_name=${SPINNAKER_APPLICATION} \
                                    --var environment=staging
                                
                                chaos-toolkit run chaos-experiments/instance-termination.yaml \
                                    --var service_name=${SPINNAKER_APPLICATION} \
                                    --var environment=staging
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Production Deployment') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Canary 배포 시작
                    def canaryDeployment = triggerSpinnakerPipeline([
                        application: env.SPINNAKER_APPLICATION,
                        pipeline: 'Canary Deploy to Production',
                        parameters: [
                            buildNumber: env.BUILD_NUMBER,
                            canaryPercentage: 5,
                            analysisInterval: '30m'
                        ]
                    ])
                    
                    // Canary 분석 결과 대기
                    def canaryResults = waitForCanaryAnalysis(canaryDeployment.executionId)
                    
                    if (canaryResults.passed) {
                        echo "[OK] Canary analysis passed, proceeding with full deployment"
                        
                        // Full production 배포
                        triggerSpinnakerPipeline([
                            application: env.SPINNAKER_APPLICATION,
                            pipeline: 'Deploy to Production',
                            parameters: [
                                buildNumber: env.BUILD_NUMBER,
                                strategy: 'red-black'
                            ]
                        ])
                    } else {
                        error("[FAIL] Canary analysis failed: ${canaryResults.reason}")
                    }
                }
            }
        }
        
        stage('Post-Deployment Monitoring') {
            steps {
                script {
                    // 배포 후 모니터링 설정
                    enableAdvancedMonitoring([
                        application: env.SPINNAKER_APPLICATION,
                        alertThresholds: [
                            errorRate: 0.01,
                            responseTime: 2000,
                            throughputDrop: 0.1
                        ],
                        monitoringDuration: '24h'
                    ])
                    
                    // SLO 추적 시작
                    startSLOTracking([
                        application: env.SPINNAKER_APPLICATION,
                        slos: [
                            [name: 'availability', target: 99.9],
                            [name: 'latency_p99', target: 1000],
                            [name: 'error_rate', target: 0.1]
                        ]
                    ])
                }
            }
        }
    }
    
    post {
        always {
            script {
                // 배포 메트릭 수집
                collectDeploymentMetrics()
                
                // 배포 대시보드 업데이트
                updateDeploymentDashboard()
            }
        }
        success {
            script {
                // 성공 배포 알림 (Netflix 스타일)
                slackSend(
                    channel: '#deployments',
                    color: 'good',
                    message: """
                        [DEPLOY OK] Netflix-style deployment successful
                        
                        Service: ${env.SPINNAKER_APPLICATION}
                        Build: #${env.BUILD_NUMBER}
                        Duration: ${currentBuild.durationString}
                        Status: Now streaming to production
                        
                        <${env.BUILD_URL}|View Build Details>
                    """
                )
                
                // 성공률 메트릭 업데이트
                updateSuccessRateMetrics()
            }
        }
        failure {
            script {
                // 자동 롤백 수행
                if (env.AUTOMATIC_ROLLBACK_ENABLED == 'true') {
                    performAutomaticRollback()
                }
                
                // 인시던트 대응팀 알림
                notifyIncidentResponse([
                    severity: 'high',
                    service: env.SPINNAKER_APPLICATION,
                    type: 'deployment_failure'
                ])
            }
        }
    }
}

// Netflix-style helper functions
def analyzeDependencies(applicationName) {
    // 의존성 분석 로직
    return [
        services: ['user-service', 'recommendation-service'],
        circularDependencies: null
    ]
}

def collectPerformanceBaseline(applicationName) {
    // 성능 기준선 수집
    return [
        avgResponseTime: 150,
        throughput: 1000,
        errorRate: 0.001
    ]
}

def runPerformanceTests(config) {
    // 성능 테스트 실행
    return [
        avgResponseTime: 145,
        throughput: 1050,
        errorRate: 0.0008
    ]
}

def triggerSpinnakerPipeline(config) {
    // Spinnaker API 호출
    return [executionId: 'exec-12345']
}

def waitForCanaryAnalysis(executionId) {
    // Canary 분석 결과 대기
    return [passed: true, score: 95]
}
```

### 6.2 Google — Monorepo 기반 빌드 시스템

배경: Google은 20억 줄의 코드를 단일 저장소로 관리하며, Bazel 빌드 시스템으로 증분 빌드를 돌린다.

```groovy
pipeline {
    agent none
    
    environment {
        BAZEL_CACHE_ENDPOINT = 'https://cache.company.com'
        BUILD_FARM_ENDPOINT = 'grpc://build-farm:8980'
        REMOTE_EXECUTION_ENABLED = 'true'
    }
    
    stages {
        stage('Change Impact Analysis') {
            agent { label 'change-analyzer' }
            steps {
                script {
                    // 변경 영향 분석
                    def changedFiles = sh(
                        script: '''
                            git diff --name-only HEAD~1 HEAD | \
                            grep -E "\\.(java|py|go|cc|h)$" || true
                        ''',
                        returnStdout: true
                    ).trim().split('\n').findAll { it }
                    
                    if (!changedFiles) {
                        echo "No source changes detected, skipping build"
                        currentBuild.result = 'NOT_BUILT'
                        return
                    }
                    
                    // 영향받는 타겟 식별
                    def affectedTargets = sh(
                        script: """
                            bazel query \
                                'rdeps(//..., set(${changedFiles.join(' ')}))' \
                                2>/dev/null | head -1000
                        """,
                        returnStdout: true
                    ).trim().split('\n').findAll { it }
                    
                    env.TARGETS_TO_BUILD = affectedTargets.join(' ')
                    env.CHANGE_SCOPE = affectedTargets.size() > 100 ? 'large' : 'small'
                    
                    echo "Affected targets: ${affectedTargets.size()}"
                    echo "Change scope: ${env.CHANGE_SCOPE}"
                }
            }
        }
        
        stage('Distributed Build') {
            when {
                expression { env.TARGETS_TO_BUILD && env.TARGETS_TO_BUILD != '' }
            }
            parallel {
                stage('Compile') {
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
                                  env:
                                  - name: USE_BAZEL_VERSION
                                    value: "6.0.0"
                            '''
                        }
                    }
                    steps {
                        container('bazel') {
                            script {
                                // 분산 빌드 실행
                                def buildStrategy = env.CHANGE_SCOPE == 'large' ? 'remote' : 'local'
                                
                                if (buildStrategy == 'remote') {
                                    sh """
                                        bazel build \
                                            --remote_executor=${BUILD_FARM_ENDPOINT} \
                                            --remote_cache=${BAZEL_CACHE_ENDPOINT} \
                                            --jobs=200 \
                                            --experimental_remote_execution_keepalive \
                                            --incompatible_strict_action_env \
                                            ${env.TARGETS_TO_BUILD}
                                    """
                                } else {
                                    sh """
                                        bazel build \
                                            --remote_cache=${BAZEL_CACHE_ENDPOINT} \
                                            --jobs=8 \
                                            ${env.TARGETS_TO_BUILD}
                                    """
                                }
                                
                                // 빌드 통계 수집
                                sh '''
                                    bazel info used-heap-size-after-gc > build-stats.txt
                                    bazel info peak-heap-size >> build-stats.txt
                                    bazel info committed-heap-size >> build-stats.txt
                                '''
                            }
                        }
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'build-stats.txt', allowEmptyArchive: true
                        }
                    }
                }
                
                stage('Test Sharding') {
                    agent {
                        kubernetes {
                            yaml '''
                              apiVersion: v1
                              kind: Pod
                              spec:
                                containers:
                                - name: bazel-test
                                  image: gcr.io/bazel-public/bazel:latest
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
                    steps {
                        container('bazel-test') {
                            script {
                                // 테스트 대상 결정
                                def testTargets = sh(
                                    script: """
                                        bazel query \
                                            'tests(${env.TARGETS_TO_BUILD})' \
                                            2>/dev/null || echo ""
                                    """,
                                    returnStdout: true
                                ).trim()
                                
                                if (testTargets) {
                                    sh """
                                        # 테스트 샤딩 실행
                                        bazel test \
                                            --remote_executor=${BUILD_FARM_ENDPOINT} \
                                            --test_output=errors \
                                            --flaky_test_attempts=3 \
                                            --runs_per_test=1 \
                                            --test_strategy=remote \
                                            --experimental_split_xml_generation \
                                            ${testTargets}
                                    """
                                } else {
                                    echo "No tests to run for affected targets"
                                }
                            }
                        }
                    }
                    post {
                        always {
                            // 테스트 결과 발행
                            publishTestResults testResultsPattern: 'bazel-testlogs/**/test.xml'
                        }
                    }
                }
            }
        }
        
        stage('Code Analysis at Scale') {
            parallel {
                stage('Static Analysis') {
                    agent { docker { image 'sonarsource/sonar-scanner-cli:latest' } }
                    steps {
                        script {
                            // 변경된 파일만 분석
                            def changedFiles = sh(
                                script: 'git diff --name-only HEAD~1 HEAD | tr "\\n" ","',
                                returnStdout: true
                            ).trim()
                            
                            if (changedFiles) {
                                sh """
                                    sonar-scanner \
                                        -Dsonar.projectKey=monorepo \
                                        -Dsonar.sources=. \
                                        -Dsonar.inclusions="${changedFiles}" \
                                        -Dsonar.analysis.mode=incremental \
                                        -Dsonar.scm.disabled=false
                                """
                            }
                        }
                    }
                }
                
                stage('Dependency Analysis') {
                    agent { label 'dependency-analyzer' }
                    steps {
                        sh '''
                            # 의존성 그래프 분석
                            bazel query \
                                --output=graph \
                                'deps(${TARGETS_TO_BUILD})' \
                                > dependency-graph.dot
                            
                            # 의존성 사이클 검출
                            bazel query \
                                --output=textproto \
                                'somepath(${TARGETS_TO_BUILD}, ${TARGETS_TO_BUILD})' \
                                > cycle-analysis.txt || true
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'dependency-graph.dot,cycle-analysis.txt', allowEmptyArchive: true
                        }
                    }
                }
            }
        }
        
        stage('Integration Testing') {
            when {
                expression { env.CHANGE_SCOPE == 'large' }
            }
            steps {
                script {
                    // 대규모 변경 시 통합 테스트 실행
                    sh '''
                        # 통합 테스트 환경 준비
                        bazel run //integration-tests:setup-environment
                        
                        # 통합 테스트 실행
                        bazel test \
                            --test_tag_filters=integration \
                            --test_timeout=1200 \
                            --test_strategy=exclusive \
                            //integration-tests/...
                    '''
                }
            }
        }
        
        stage('Performance Benchmarking') {
            when {
                anyOf {
                    expression { env.CHANGE_SCOPE == 'large' }
                    changeset "**/performance/**"
                }
            }
            steps {
                script {
                    // 성능 벤치마크 실행
                    sh '''
                        # 현재 성능 기준선 측정
                        bazel run //benchmarks:micro-benchmarks \
                            --run_under="//tools:benchmark-runner --output=current-benchmark.json"
                        
                        # 이전 기준선과 비교
                        python3 scripts/compare-benchmarks.py \
                            --baseline=previous-benchmark.json \
                            --current=current-benchmark.json \
                            --threshold=0.05 \
                            --output=benchmark-comparison.json
                    '''
                    
                    // 성능 회귀 분석
                    def benchmarkResults = readJSON file: 'benchmark-comparison.json'
                    if (benchmarkResults.regressions?.size() > 0) {
                        currentBuild.result = 'UNSTABLE'
                        echo "[WARN] Performance regressions detected: ${benchmarkResults.regressions.size()}"
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                // 빌드 캐시 통계 수집
                sh '''
                    bazel info | grep -E "(cache|remote)" > cache-stats.txt
                '''
                
                // Bazel 메트릭을 Prometheus에 전송
                sh '''
                    curl -X POST "${PROMETHEUS_PUSHGATEWAY}/metrics/job/bazel-build" \
                        --data-binary @- << EOF
bazel_build_duration_seconds{job="${JOB_NAME}"} ${BUILD_DURATION}
bazel_cache_hit_rate{job="${JOB_NAME}"} $(grep "cache-hit-rate" cache-stats.txt | awk '{print $2}')
bazel_targets_built{job="${JOB_NAME}"} $(echo "${TARGETS_TO_BUILD}" | wc -w)
EOF
                '''
            }
        }
        success {
            script {
                // 빌드 아티팩트를 아티팩트 저장소에 업로드
                if (env.BRANCH_NAME == 'main') {
                    sh '''
                        # 릴리스 아티팩트 생성
                        bazel build //release:all --compilation_mode=opt
                        
                        # 아티팩트 업로드
                        bazel run //tools:upload-artifacts -- \
                            --version=${BUILD_NUMBER} \
                            --repository=releases
                    '''
                }
            }
        }
    }
}
```

## 7. 실전 프로젝트 — 종합 실습

### 7.1 초급 실습 — 환경 변수 활용

목표: 변하는 환경 변수와 조건부 로직으로 파이프라인을 구성한다.

요구사항:
1. Git 브랜치에 따른 환경 자동 결정
2. 빌드 도구 자동 감지 (package.json 또는 pom.xml)
3. 병렬 품질 검증 (테스트, 린팅, 보안 스캔)
4. 환경별 배포 전략

실습 템플릿:
```groovy
pipeline {
    agent any
    
    environment {
        // 동적 환경 변수 구성
        BUILD_TOOL = script {
            if (fileExists('package.json')) {
                return 'npm'
            } else if (fileExists('pom.xml')) {
                return 'maven'
            } else if (fileExists('build.gradle')) {
                return 'gradle'
            } else {
                return 'unknown'
            }
        }
        
        DEPLOY_ENV = script {
            switch(env.BRANCH_NAME) {
                case 'main':
                case 'master':
                    return 'production'
                case 'develop':
                    return 'staging'
                case ~/^feature\/.*/:
                    return 'development'
                default:
                    return 'review'
            }
        }
        
        // TODO: 여러분의 팀명으로 변경하세요
        TEAM_NAME = 'YOUR_TEAM_NAME'
        
        // 동적 빌드 정보
        BUILD_VERSION = "${BUILD_NUMBER}-${GIT_COMMIT.take(8)}"
        BUILD_TIMESTAMP = sh(
            script: 'date +%Y%m%d-%H%M%S',
            returnStdout: true
        ).trim()
    }
    
    stages {
        stage('Environment Info') {
            steps {
                script {
                    echo """
                        [START] 파이프라인 시작
                        ==================
                        팀명: ${env.TEAM_NAME}
                        빌드 도구: ${env.BUILD_TOOL}
                        배포 환경: ${env.DEPLOY_ENV}
                        빌드 버전: ${env.BUILD_VERSION}
                        빌드 시간: ${env.BUILD_TIMESTAMP}
                        브랜치: ${env.BRANCH_NAME}
                        ==================
                    """
                }
            }
        }
        
        // TODO: 빌드 스테이지 구현
        stage('Build') {
            steps {
                script {
                    switch(env.BUILD_TOOL) {
                        case 'npm':
                            sh 'npm ci && npm run build'
                            break
                        case 'maven':
                            sh 'mvn clean package -DskipTests'
                            break
                        case 'gradle':
                            sh './gradlew clean build -x test'
                            break
                        default:
                            echo 'Unknown build tool, skipping build'
                    }
                }
            }
        }
        
        // TODO: 병렬 품질 검증 구현
        stage('Quality Gates') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        // 여기에 테스트 로직 구현
                        echo 'Running unit tests...'
                    }
                }
                stage('Linting') {
                    steps {
                        // 여기에 린팅 로직 구현  
                        echo 'Running code linting...'
                    }
                }
                stage('Security Scan') {
                    steps {
                        // 여기에 보안 스캔 로직 구현
                        echo 'Running security scan...'
                    }
                }
            }
        }
        
        // TODO: 조건부 배포 구현
        stage('Deploy') {
            when {
                // 조건 구현: main/develop 브랜치에서만 배포
            }
            steps {
                script {
                    echo "[DEPLOY] ${env.TEAM_NAME} -> ${env.DEPLOY_ENV}"
                    
                    // 환경별 배포 로직
                    switch(env.DEPLOY_ENV) {
                        case 'production':
                            echo 'Deploying to production with blue-green strategy'
                            break
                        case 'staging':
                            echo 'Deploying to staging environment'
                            break
                        default:
                            echo "Deploying to ${env.DEPLOY_ENV} environment"
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo "파이프라인 완료: ${currentBuild.currentResult}"
        }
        success {
            echo "[OK] ${env.TEAM_NAME} 팀의 빌드가 성공했습니다."
        }
        failure {
            echo "[FAIL] ${env.TEAM_NAME} 팀의 빌드가 실패했습니다."
        }
    }
}
```

### 7.2 중급 실습 — 매트릭스 빌드와 스테이지 자동 생성

목표: 여러 환경, 여러 버전에서 매트릭스 빌드를 만든다.

실습 과제:
```groovy
pipeline {
    agent none
    
    stages {
        stage('Matrix Testing') {
            matrix {
                axes {
                    axis {
                        name 'NODE_VERSION'
                        values '16', '18', '20'
                    }
                    axis {
                        name 'ENVIRONMENT'
                        values 'ubuntu-latest', 'alpine'
                    }
                }
                stages {
                    stage('Test Matrix') {
                        agent {
                            docker {
                                image "node:${NODE_VERSION}-${ENVIRONMENT}"
                            }
                        }
                        steps {
                            script {
                                // TODO: 매트릭스별 테스트 구현
                                echo "Testing on Node ${NODE_VERSION} with ${ENVIRONMENT}"
                                
                                // 환경별 특화 로직 구현하기
                            }
                        }
                    }
                }
            }
        }
    }
}
```

### 7.3 고급 실습 — 운영 환경에서 굴리는 파이프라인

목표: 실제 기업 환경에서 그대로 굴릴 수 있는 완전한 파이프라인을 만든다.

요구사항:
- 보안 스캔과 컴플라이언스 체크
- 자동 롤백 기능
- 실시간 모니터링과 알림
- 승인 워크플로우
- 메트릭 수집과 리포팅

## 8. 문제 해결 가이드

### 8.1 성능 문제 진단 및 해결

병렬 처리 최적화 체크리스트:
```groovy
pipeline {
    agent any
    
    stages {
        stage('Performance Diagnosis') {
            steps {
                script {
                    // 1. 시스템 리소스 확인
                    sh '''
                        echo "=== 시스템 리소스 현황 ==="
                        free -h
                        nproc
                        df -h
                        iostat -x 1 3 || echo "iostat not available"
                    '''
                    
                    // 2. 네트워크 대역폭 테스트
                    sh '''
                        echo "=== 네트워크 성능 테스트 ==="
                        time curl -s -o /dev/null https://httpbin.org/delay/1
                        ping -c 3 google.com || echo "Network connectivity issue"
                    '''
                    
                    // 3. 빌드 도구 성능 설정
                    sh '''
                        echo "=== 빌드 도구 설정 확인 ==="
                        java -version 2>&1 | head -3
                        mvn -version | head -3 || echo "Maven not available"
                        echo "MAVEN_OPTS: $MAVEN_OPTS"
                    '''
                }
            }
        }
    }
}
```

### 8.2 일반적인 오류와 해결방법

매트릭스 빌드 실패 대응:
```groovy
stage('Matrix Build with Error Handling') {
    matrix {
        axes {
            axis {
                name 'PLATFORM'
                values 'linux', 'windows', 'macos'
            }
        }
        stages {
            stage('Build with Retry') {
                steps {
                    retry(3) {
                        script {
                            try {
                                sh "echo 'Building for ${PLATFORM}'"
                                // 실제 빌드 로직
                                
                            } catch (Exception e) {
                                echo "Build attempt failed: ${e.getMessage()}"
                                
                                // 에러 유형별 대응
                                if (e.getMessage().contains('timeout')) {
                                    echo "Timeout error detected, increasing timeout"
                                    sleep 30
                                } else if (e.getMessage().contains('network')) {
                                    echo "Network error detected, waiting for recovery"
                                    sleep 60
                                }
                                
                                throw e
                            }
                        }
                    }
                }
            }
        }
    }
}
```

## 9. 최종 평가 및 다음 단계

### 9.1 Self-Assessment Quiz (고급)

Q1. 고급 환경 변수 관리
다음 중 동적 환경 변수를 올바르게 생성하는 방법은?

A) `environment { DYNAMIC_VAR = "$(date)" }`
B) `environment { DYNAMIC_VAR = script { return new Date() } }`  
C) `environment { DYNAMIC_VAR = sh(script: 'date', returnStdout: true).trim() }`
D) `script { env.DYNAMIC_VAR = sh(script: 'date', returnStdout: true).trim() }`

정답: C
해설: environment 블록에서는 `sh()` 함수를 써서 동적 값을 만들 수 있다.

Q2. 병렬 처리 최적화  
100개의 테스트를 가장 효율적으로 병렬 실행하려면?

A) 100개의 parallel 블록 생성
B) matrix 빌드로 축 분할
C) script 블록에서 동적 parallel 태스크 생성
D) 모든 테스트를 하나의 stage에서 순차 실행

정답: C  
해설: 동적으로 parallel 태스크를 만들면 리소스 효율과 실행 시간을 함께 잡는다.

Q3. 보안 파이프라인 설계
운영 환경 보안 파이프라인의 필수 구성 요소는?

A) SAST만 수행
B) SAST + SCA + 시크릿 스캔 + 컨테이너 스캔 + 컴플라이언스 체크  
C) 코드 리뷰만 수행
D) 수동 보안 검토만 수행

정답: B
해설: 여러 단계에서 검증을 걸어야 한 곳이 뚫려도 다음 단계에서 잡힌다.

### 9.2 다음 주차 예고

5주차: Docker 통합과 컨테이너 기반 파이프라인
- Docker-in-Docker (DinD) 고급 구성
- 멀티 스테이지 빌드 최적화  
- 컨테이너 레지스트리 보안
- Kubernetes 기반 동적 에이전트
- 컨테이너 취약점 스캔 자동화

지금까지 다룬 고급 파이프라인 기법과 Docker가 만나면 컨테이너 기반 CI/CD가 손에 잡힌다. 다음 주에는 컨테이너를 파이프라인 안쪽까지 끌고 들어오는 방법을 본다.

4주를 마치며 매트릭스 빌드, shared library, canary 배포까지 훑었다. 5주차는 컨테이너 통합이다.
