---
layout: post
title: "DevSecOps Mastery: 6주차 - 지속적 통합 실무와 품질 게이트"
date: 2026-05-20 16:45:00 +0900
---

# DevSecOps Mastery: 6주차 - 지속적 통합 실무와 품질 게이트

## 서론: 빌드 성공이 곧 배포 가능함을 의미하는가?

지난 5주간 우리는 Jenkins 파이프라인을 통해 코드를 자동으로 빌드하고 컨테이너로 패키징하는 방법을 익혔습니다. 하지만 여기서 중요한 질문이 하나 남습니다.

**"빌드가 성공했다고 해서 정말 배포해도 괜찮을까요?"**

### 현실에서 벌어지는 일들

실제 기업 환경에서 관찰되는 문제들을 살펴보겠습니다.

**Spotify의 2017년 사고 사례**:
한 개발팀이 음악 재생 알고리즘을 개선하는 코드를 배포했습니다. Jenkins 빌드는 완벽하게 성공했고, 모든 단위 테스트도 통과했습니다. 하지만 배포 후 30분 만에 전체 플랫폼의 음악 재생이 중단되었습니다.

**원인**: 새로운 알고리즘이 대용량 플레이리스트에 대해 예상보다 100배 많은 CPU를 사용했습니다. 단위 테스트는 작은 데이터셋으로만 검증했기 때문에 문제를 발견하지 못했습니다.

**결과**: 
- 2,500만 사용자에게 30분간 서비스 중단
- 잃어버린 매출: 약 50만 달러
- 고객 신뢰도 하락과 경쟁사 이탈

**Facebook의 2019년 Instagram 통합 사고**:
Instagram과 Facebook 메시징 통합 작업 중, 모든 코드 리뷰와 단위 테스트를 통과한 코드가 프로덕션에서 예기치 못한 무한 루프를 발생시켰습니다.

**교훈**: 빌드 성공 ≠ 프로덕션 준비 완료

진정한 CI(Continuous Integration)는 단순히 코드를 합치는 것이 아닙니다. **"언제든 배포 가능한 상태로 코드를 유지하는 것"**입니다.

오늘은 빌드 성공을 넘어 **배포 가능한 품질**을 보장하는 포괄적인 CI 실무를 학습합니다.

## 1. 포괄적 테스트 전략과 자동화

### 1.1 Testing Pyramid의 실무적 구현

**전통적 Testing Pyramid vs 현대적 Testing Diamond**:

```bash
# 전통적 Testing Pyramid
        /\
       /UI\     ← 소수의 End-to-End 테스트
      /____\    
     /      \   
    /Integration\ ← 적당한 통합 테스트
   /____________\
  /              \
 /  Unit Tests    \ ← 대부분의 단위 테스트
/_________________\

# 현대적 Testing Diamond (Microservices 시대)
        /\
       /E2E\    ← Critical User Journey 중심
      /____\    
     /      \   
    /Contract \ ← API 계약 테스트 (중요!)
   /__ Tests__\
  /            \
 /Integration  \ ← 서비스 간 통신 검증
/_____________/
/              \
|  Unit Tests   | ← 비즈니스 로직 검증
\______________/
```

### 1.2 다층 테스트 자동화 구현

**종합적 테스트 파이프라인**:

```groovy
pipeline {
    agent any
    
    environment {
        // 테스트 환경 설정
        TEST_DATABASE_URL = 'jdbc:h2:mem:testdb'
        INTEGRATION_TEST_PORT = '8080'
        E2E_TEST_URL = 'http://localhost:8080'
        
        // 품질 게이트 임계값
        UNIT_TEST_COVERAGE_THRESHOLD = '80'
        INTEGRATION_TEST_THRESHOLD = '90'
        PERFORMANCE_THRESHOLD = '2000' // ms
    }
    
    stages {
        stage('Parallel Testing Strategy') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh '''
                            echo "=== Unit Test Execution ==="
                            
                            # 단위 테스트 실행 (빠른 피드백)
                            npm run test:unit -- --coverage --reporter=junit --outputFile=unit-test-results.xml
                            
                            # 커버리지 검증
                            COVERAGE=$(npm run test:coverage:check | grep "Lines" | awk '{print $3}' | sed 's/%//')
                            echo "Code coverage: ${COVERAGE}%"
                            
                            if [ "$COVERAGE" -lt "$UNIT_TEST_COVERAGE_THRESHOLD" ]; then
                                echo "❌ Code coverage ($COVERAGE%) below threshold ($UNIT_TEST_COVERAGE_THRESHOLD%)"
                                exit 1
                            fi
                            
                            echo "✅ Unit tests passed with ${COVERAGE}% coverage"
                        '''
                    }
                    post {
                        always {
                            publishTestResults testResultsPattern: 'unit-test-results.xml'
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'coverage',
                                reportFiles: 'index.html',
                                reportName: 'Unit Test Coverage Report'
                            ])
                        }
                    }
                }
                
                stage('Static Code Analysis') {
                    steps {
                        sh '''
                            echo "=== Static Code Analysis ==="
                            
                            # ESLint 정적 분석
                            npm run lint -- --format=junit --output-file=eslint-results.xml
                            
                            # SonarQube 분석 (옵션)
                            if [ -n "$SONARQUBE_TOKEN" ]; then
                                sonar-scanner \
                                    -Dsonar.projectKey=myapp \
                                    -Dsonar.sources=src \
                                    -Dsonar.host.url=$SONAR_HOST_URL \
                                    -Dsonar.login=$SONARQUBE_TOKEN
                            fi
                            
                            # 코드 복잡도 검사
                            npm run complexity:check
                            
                            echo "✅ Static analysis completed"
                        '''
                    }
                    post {
                        always {
                            publishTestResults testResultsPattern: 'eslint-results.xml'
                        }
                    }
                }
                
                stage('Security Scanning') {
                    steps {
                        sh '''
                            echo "=== Security Vulnerability Scanning ==="
                            
                            # npm audit 보안 스캔
                            npm audit --audit-level=high --json > security-audit.json || true
                            
                            # 고위험 취약점 확인
                            HIGH_VULNS=$(cat security-audit.json | jq '.metadata.vulnerabilities.high // 0')
                            CRITICAL_VULNS=$(cat security-audit.json | jq '.metadata.vulnerabilities.critical // 0')
                            
                            echo "Security scan results:"
                            echo "- Critical vulnerabilities: $CRITICAL_VULNS"
                            echo "- High vulnerabilities: $HIGH_VULNS"
                            
                            if [ "$CRITICAL_VULNS" -gt 0 ]; then
                                echo "❌ Critical vulnerabilities found, blocking deployment"
                                exit 1
                            fi
                            
                            if [ "$HIGH_VULNS" -gt 5 ]; then
                                echo "⚠️ Too many high vulnerabilities ($HIGH_VULNS), review required"
                                exit 1
                            fi
                            
                            echo "✅ Security scan passed"
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'security-audit.json'
                        }
                    }
                }
            }
        }
        
        stage('Integration Testing') {
            steps {
                sh '''
                    echo "=== Integration Test Setup ==="
                    
                    # 테스트용 데이터베이스 및 서비스 시작
                    docker-compose -f docker-compose.test.yml up -d
                    
                    # 서비스 준비 대기
                    timeout=60
                    while [ $timeout -gt 0 ]; do
                        if curl -f http://localhost:$INTEGRATION_TEST_PORT/health; then
                            echo "✅ Application is ready for integration testing"
                            break
                        fi
                        echo "Waiting for application to start... ($timeout seconds remaining)"
                        sleep 2
                        timeout=$((timeout - 2))
                    done
                    
                    if [ $timeout -le 0 ]; then
                        echo "❌ Application failed to start within timeout"
                        exit 1
                    fi
                    
                    # 통합 테스트 실행
                    npm run test:integration -- --reporter=junit --outputFile=integration-test-results.xml
                    
                    echo "✅ Integration tests completed"
                '''
            }
            post {
                always {
                    sh '''
                        # 테스트 환경 정리
                        docker-compose -f docker-compose.test.yml down || true
                    '''
                    publishTestResults testResultsPattern: 'integration-test-results.xml'
                }
            }
        }
        
        stage('Contract Testing') {
            steps {
                sh '''
                    echo "=== API Contract Testing ==="
                    
                    # Pact 계약 테스트 실행
                    npm run test:contract
                    
                    # API 스펙 검증
                    if [ -f "api-spec.yaml" ]; then
                        swagger-codegen validate -i api-spec.yaml
                        echo "✅ API specification is valid"
                    fi
                    
                    # 스키마 호환성 검증 (GraphQL/gRPC)
                    if [ -f "schema.graphql" ]; then
                        npm run schema:validate
                        echo "✅ Schema compatibility verified"
                    fi
                    
                    echo "✅ Contract tests passed"
                '''
            }
        }
        
        stage('Performance Testing') {
            steps {
                sh '''
                    echo "=== Performance Testing ==="
                    
                    # 애플리케이션이 실행 중인지 확인
                    if ! curl -f http://localhost:$INTEGRATION_TEST_PORT/health; then
                        echo "❌ Application not running for performance tests"
                        exit 1
                    fi
                    
                    # 부하 테스트 실행 (Artillery 사용)
                    cat > performance-test.yml << EOF
                    config:
                      target: 'http://localhost:${INTEGRATION_TEST_PORT}'
                      phases:
                        - duration: 60
                          arrivalRate: 10
                        - duration: 120
                          arrivalRate: 50
                        - duration: 60
                          arrivalRate: 100
                      processor: "./performance-processor.js"
                    scenarios:
                      - name: "API Load Test"
                        flow:
                          - get:
                              url: "/api/health"
                          - get:
                              url: "/api/users"
                          - post:
                              url: "/api/users"
                              json:
                                name: "Test User"
                                email: "test@example.com"
                    EOF
                    
                    # 성능 테스트 실행
                    npx artillery run performance-test.yml --output performance-results.json
                    
                    # 결과 분석
                    MEDIAN_RESPONSE_TIME=$(cat performance-results.json | jq '.aggregate.latency.median')
                    P95_RESPONSE_TIME=$(cat performance-results.json | jq '.aggregate.latency.p95')
                    
                    echo "Performance test results:"
                    echo "- Median response time: ${MEDIAN_RESPONSE_TIME}ms"
                    echo "- P95 response time: ${P95_RESPONSE_TIME}ms"
                    
                    # 성능 임계값 검증
                    if [ $(echo "$P95_RESPONSE_TIME > $PERFORMANCE_THRESHOLD" | bc -l) -eq 1 ]; then
                        echo "❌ P95 response time (${P95_RESPONSE_TIME}ms) exceeds threshold (${PERFORMANCE_THRESHOLD}ms)"
                        exit 1
                    fi
                    
                    echo "✅ Performance tests passed"
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'performance-results.json'
                }
            }
        }
        
        stage('End-to-End Testing') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    changeRequest()
                }
            }
            steps {
                sh '''
                    echo "=== End-to-End Testing ==="
                    
                    # Cypress E2E 테스트 실행
                    npm run test:e2e:headless -- --reporter junit --reporter-options "outputFile=e2e-test-results.xml"
                    
                    # 크리티컬 유저 저니 검증
                    npm run test:critical-path
                    
                    echo "✅ E2E tests completed"
                '''
            }
            post {
                always {
                    publishTestResults testResultsPattern: 'e2e-test-results.xml'
                    // 스크린샷 및 비디오 아티팩트 보관
                    archiveArtifacts artifacts: 'cypress/screenshots/**/*.png, cypress/videos/**/*.mp4', allowEmptyArchive: true
                }
            }
        }
    }
    
    post {
        always {
            // 모든 테스트 결과 통합 리포트
            publishTestResults testResultsPattern: '*-test-results.xml'
        }
    }
}
```

### 1.3 Test-Driven Development (TDD) 통합

**Red-Green-Refactor 사이클의 CI 통합**:

```groovy
pipeline {
    agent any
    
    environment {
        TDD_MODE = 'enabled'
    }
    
    stages {
        stage('TDD Validation') {
            when {
                changeRequest()
            }
            steps {
                script {
                    // PR에서 TDD 패턴 검증
                    sh '''
                        echo "=== TDD Pattern Validation ==="
                        
                        # 새로 추가된 테스트 파일 확인
                        NEW_TESTS=$(git diff --name-only origin/main...HEAD | grep -E "\\.(test|spec)\\." | wc -l)
                        
                        # 새로 추가된 소스 파일 확인
                        NEW_SOURCE=$(git diff --name-only origin/main...HEAD | grep -v -E "\\.(test|spec)\\." | grep -E "\\.(js|ts|py|java)$" | wc -l)
                        
                        echo "New test files: $NEW_TESTS"
                        echo "New source files: $NEW_SOURCE"
                        
                        # TDD 규칙 검증: 테스트가 먼저 작성되었는가?
                        if [ "$NEW_SOURCE" -gt 0 ] && [ "$NEW_TESTS" -eq 0 ]; then
                            echo "⚠️ Warning: Source code added without corresponding tests"
                            echo "Consider following TDD practices"
                        fi
                        
                        # 테스트 커버리지 변화 확인
                        npm run test:coverage:diff
                    '''
                }
            }
        }
        
        stage('Red Phase Validation') {
            steps {
                sh '''
                    echo "=== Red Phase: Ensure Tests Fail First ==="
                    
                    # 새로운 테스트가 실제로 실패하는지 확인 (선택적)
                    # 이를 통해 테스트가 실제 기능을 검증하는지 확인
                    
                    if [ "$TDD_MODE" = "enabled" ]; then
                        echo "TDD mode enabled - validating test effectiveness"
                    fi
                '''
            }
        }
    }
}
```

## 2. 코드 품질 게이트와 메트릭

### 2.1 SonarQube 통합 품질 게이트

**종합적 코드 품질 분석**:

```groovy
pipeline {
    agent any
    
    environment {
        SONAR_SCANNER_HOME = tool 'SonarQubeScanner'
        SONAR_PROJECT_KEY = 'myapp'
        SONAR_PROJECT_NAME = 'My Application'
    }
    
    stages {
        stage('Code Quality Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        echo "=== SonarQube Code Analysis ==="
                        
                        $SONAR_SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                            -Dsonar.projectName="$SONAR_PROJECT_NAME" \
                            -Dsonar.projectVersion=${BUILD_NUMBER} \
                            -Dsonar.sources=src \
                            -Dsonar.tests=test \
                            -Dsonar.language=js \
                            -Dsonar.sourceEncoding=UTF-8 \
                            -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                            -Dsonar.testExecutionReportPaths=test-results.xml \
                            -Dsonar.coverage.exclusions="**/*.test.js,**/*.spec.js,**/node_modules/**" \
                            -Dsonar.cpd.exclusions="**/*.test.js,**/*.spec.js"
                    '''
                }
            }
        }
        
        stage('Quality Gate Validation') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    script {
                        def qualityGate = waitForQualityGate()
                        
                        echo "Quality Gate Status: ${qualityGate.status}"
                        
                        if (qualityGate.status != 'OK') {
                            echo "❌ Quality Gate Failed!"
                            echo "Quality Gate Details:"
                            
                            // 상세한 실패 이유 출력
                            qualityGate.conditions.each { condition ->
                                echo "- ${condition.metricKey}: ${condition.actualValue} (threshold: ${condition.errorThreshold})"
                            }
                            
                            error "Quality Gate Failed: ${qualityGate.status}"
                        }
                        
                        echo "✅ Quality Gate Passed"
                    }
                }
            }
        }
        
        stage('Custom Quality Metrics') {
            steps {
                sh '''
                    echo "=== Custom Quality Metrics Analysis ==="
                    
                    # 코드 복잡도 분석
                    npx complexity-report src/ --format json --output complexity-report.json
                    
                    # 기술 부채 측정
                    npx jscpd src/ --format json --output duplication-report.json
                    
                    # 의존성 분석
                    npm audit --json > dependency-audit.json || true
                    
                    # 메트릭 분석 및 임계값 검증
                    node << 'EOF'
                    const complexityReport = require('./complexity-report.json');
                    const duplicationReport = require('./duplication-report.json');
                    
                    // 복잡도 임계값 검증
                    const avgComplexity = complexityReport.reports.reduce((sum, report) => 
                        sum + report.complexity.aggregate.complexity, 0) / complexityReport.reports.length;
                    
                    console.log(`Average complexity: ${avgComplexity}`);
                    
                    if (avgComplexity > 10) {
                        console.log('❌ Average complexity too high');
                        process.exit(1);
                    }
                    
                    // 중복 코드 임계값 검증
                    const duplicationPercentage = duplicationReport.statistics.total.percentage;
                    console.log(`Code duplication: ${duplicationPercentage}%`);
                    
                    if (duplicationPercentage > 15) {
                        console.log('❌ Code duplication too high');
                        process.exit(1);
                    }
                    
                    console.log('✅ Custom quality metrics passed');
                    EOF
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: '*-report.json'
                }
            }
        }
    }
}
```

### 2.2 동적 품질 게이트 설정

**프로젝트 단계별 적응형 품질 기준**:

```groovy
pipeline {
    agent any
    
    environment {
        // 프로젝트 성숙도에 따른 품질 기준
        PROJECT_MATURITY = getProjectMaturity()
        QUALITY_PROFILE = getQualityProfile()
    }
    
    stages {
        stage('Adaptive Quality Gates') {
            steps {
                script {
                    def qualityStandards = [:]
                    
                    // 프로젝트 단계별 품질 기준 설정
                    switch(env.PROJECT_MATURITY) {
                        case 'prototype':
                            qualityStandards = [
                                coverage: 50,
                                complexity: 15,
                                duplication: 20,
                                vulnerabilities: 'medium'
                            ]
                            break
                        case 'mvp':
                            qualityStandards = [
                                coverage: 70,
                                complexity: 12,
                                duplication: 15,
                                vulnerabilities: 'low'
                            ]
                            break
                        case 'production':
                            qualityStandards = [
                                coverage: 85,
                                complexity: 10,
                                duplication: 10,
                                vulnerabilities: 'none'
                            ]
                            break
                    }
                    
                    echo "Quality standards for ${env.PROJECT_MATURITY} project:"
                    echo "- Code coverage: ${qualityStandards.coverage}%"
                    echo "- Max complexity: ${qualityStandards.complexity}"
                    echo "- Max duplication: ${qualityStandards.duplication}%"
                    echo "- Vulnerability tolerance: ${qualityStandards.vulnerabilities}"
                    
                    // 동적 검증 수행
                    validateQualityStandards(qualityStandards)
                }
            }
        }
    }
}

def getProjectMaturity() {
    if (env.BRANCH_NAME.startsWith('prototype/')) {
        return 'prototype'
    } else if (env.BRANCH_NAME == 'develop' || env.BRANCH_NAME.startsWith('feature/')) {
        return 'mvp'
    } else if (env.BRANCH_NAME == 'main') {
        return 'production'
    }
    return 'mvp'
}

def getQualityProfile() {
    def maturity = getProjectMaturity()
    return "quality-profile-${maturity}"
}

def validateQualityStandards(standards) {
    sh """
        echo "=== Dynamic Quality Validation ==="
        
        # 커버리지 검증
        COVERAGE=\$(npm run test:coverage:percentage)
        if [ "\$COVERAGE" -lt "${standards.coverage}" ]; then
            echo "❌ Coverage (\$COVERAGE%) below ${standards.coverage}%"
            exit 1
        fi
        
        # 복잡도 검증
        MAX_COMPLEXITY=\$(npx complexity-report src/ --format json | jq '.reports | max_by(.complexity.aggregate.complexity) | .complexity.aggregate.complexity')
        if [ "\$MAX_COMPLEXITY" -gt "${standards.complexity}" ]; then
            echo "❌ Complexity (\$MAX_COMPLEXITY) above ${standards.complexity}"
            exit 1
        fi
        
        echo "✅ Quality standards met for ${env.PROJECT_MATURITY} project"
    """
}
```

## 3. 고급 아티팩트 관리와 배포 준비

### 3.1 Semantic Versioning과 자동 태깅

**자동화된 버전 관리 시스템**:

```groovy
pipeline {
    agent any
    
    environment {
        // Semantic versioning 설정
        MAJOR_VERSION = '1'
        MINOR_VERSION = getMinorVersion()
        PATCH_VERSION = env.BUILD_NUMBER
        SEMANTIC_VERSION = "${MAJOR_VERSION}.${MINOR_VERSION}.${PATCH_VERSION}"
        
        // 아티팩트 저장소 설정
        ARTIFACTORY_URL = 'https://artifactory.company.com'
        DOCKER_REGISTRY = 'registry.company.com'
        NPM_REGISTRY = 'https://npm.company.com'
    }
    
    stages {
        stage('Version Management') {
            steps {
                script {
                    // Conventional Commits 기반 버전 결정
                    def versionBump = determineVersionBump()
                    def newVersion = calculateNewVersion(versionBump)
                    
                    env.CALCULATED_VERSION = newVersion
                    echo "Calculated version: ${newVersion}"
                    
                    // package.json 버전 업데이트
                    sh """
                        echo "=== Version Management ==="
                        
                        # package.json 버전 업데이트
                        npm version ${newVersion} --no-git-tag-version
                        
                        # Git 태그 생성 준비
                        echo "${newVersion}" > VERSION
                        
                        echo "✅ Version set to ${newVersion}"
                    """
                }
            }
        }
        
        stage('Multi-Format Artifact Creation') {
            parallel {
                stage('NPM Package') {
                    steps {
                        sh '''
                            echo "=== NPM Package Creation ==="
                            
                            # 의존성 최적화
                            npm prune --production
                            
                            # 패키지 생성
                            npm pack
                            
                            # 패키지 메타데이터 생성
                            cat > package-metadata.json << EOF
                            {
                              "name": "$(npm run env | grep npm_package_name | cut -d= -f2)",
                              "version": "${CALCULATED_VERSION}",
                              "buildNumber": "${BUILD_NUMBER}",
                              "gitCommit": "${GIT_COMMIT}",
                              "buildDate": "$(date -Iseconds)",
                              "nodeVersion": "$(node --version)",
                              "npmVersion": "$(npm --version)"
                            }
                            EOF
                            
                            echo "✅ NPM package created"
                        '''
                    }
                    post {
                        success {
                            archiveArtifacts artifacts: '*.tgz, package-metadata.json'
                        }
                    }
                }
                
                stage('Docker Image') {
                    steps {
                        sh '''
                            echo "=== Docker Image Creation ==="
                            
                            # 멀티 스테이지 빌드로 최적화된 이미지 생성
                            cat > Dockerfile.production << 'EOF'
                            # Build stage
                            FROM node:18-alpine AS builder
                            WORKDIR /app
                            COPY package*.json ./
                            RUN npm ci --only=production && npm cache clean --force
                            
                            # Production stage
                            FROM node:18-alpine AS production
                            RUN addgroup -g 1001 -S nodejs && \
                                adduser -S nextjs -u 1001
                            
                            WORKDIR /app
                            COPY --from=builder --chown=nextjs:nodejs /app/node_modules ./node_modules
                            COPY --chown=nextjs:nodejs . .
                            
                            USER nextjs
                            EXPOSE 3000
                            
                            CMD ["npm", "start"]
                            EOF
                            
                            # 이미지 빌드
                            docker build -f Dockerfile.production -t ${DOCKER_REGISTRY}/myapp:${CALCULATED_VERSION} .
                            docker tag ${DOCKER_REGISTRY}/myapp:${CALCULATED_VERSION} ${DOCKER_REGISTRY}/myapp:latest
                            
                            # 이미지 메타데이터 추가
                            docker build -f Dockerfile.production \
                                --label "version=${CALCULATED_VERSION}" \
                                --label "build-number=${BUILD_NUMBER}" \
                                --label "git-commit=${GIT_COMMIT}" \
                                --label "build-date=$(date -Iseconds)" \
                                -t ${DOCKER_REGISTRY}/myapp:${CALCULATED_VERSION} .
                            
                            echo "✅ Docker image created"
                        '''
                    }
                }
                
                stage('Static Assets') {
                    steps {
                        sh '''
                            echo "=== Static Assets Packaging ==="
                            
                            # 프로덕션 빌드
                            npm run build
                            
                            # 정적 자산 압축
                            tar -czf static-assets-${CALCULATED_VERSION}.tar.gz dist/
                            
                            # 자산 매니페스트 생성
                            cat > assets-manifest.json << EOF
                            {
                              "version": "${CALCULATED_VERSION}",
                              "buildNumber": "${BUILD_NUMBER}",
                              "assets": [
                                $(find dist/ -type f -name "*.js" -o -name "*.css" -o -name "*.html" | jq -R . | tr '\n' ',' | sed 's/,$//')
                              ],
                              "totalSize": "$(du -sh dist/ | cut -f1)",
                              "buildDate": "$(date -Iseconds)"
                            }
                            EOF
                            
                            echo "✅ Static assets packaged"
                        '''
                    }
                    post {
                        success {
                            archiveArtifacts artifacts: 'static-assets-*.tar.gz, assets-manifest.json'
                        }
                    }
                }
            }
        }
        
        stage('Artifact Security Scanning') {
            steps {
                sh '''
                    echo "=== Artifact Security Scanning ==="
                    
                    # NPM 패키지 스캔
                    if [ -f *.tgz ]; then
                        npm audit --package-lock-only --audit-level high
                    fi
                    
                    # Docker 이미지 스캔
                    if docker images ${DOCKER_REGISTRY}/myapp:${CALCULATED_VERSION} > /dev/null 2>&1; then
                        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                            aquasec/trivy:latest image \
                            --format json \
                            --output docker-security-scan.json \
                            ${DOCKER_REGISTRY}/myapp:${CALCULATED_VERSION}
                        
                        # 취약점 수준 확인
                        CRITICAL_VULNS=$(cat docker-security-scan.json | jq '[.Results[]? | .Vulnerabilities[]? | select(.Severity == "CRITICAL")] | length')
                        if [ "$CRITICAL_VULNS" -gt 0 ]; then
                            echo "❌ Critical vulnerabilities found in Docker image"
                            exit 1
                        fi
                    fi
                    
                    echo "✅ Artifact security scanning completed"
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: '*-security-scan.json', allowEmptyArchive: true
                }
            }
        }
        
        stage('Artifact Publishing') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    parallel(
                        "Publish NPM": {
                            withCredentials([string(credentialsId: 'npm-token', variable: 'NPM_TOKEN')]) {
                                sh '''
                                    echo "=== NPM Package Publishing ==="
                                    
                                    # NPM 레지스트리 인증
                                    echo "//npm.company.com/:_authToken=${NPM_TOKEN}" > ~/.npmrc
                                    
                                    # 패키지 발행
                                    npm publish --registry=${NPM_REGISTRY}
                                    
                                    echo "✅ NPM package published"
                                '''
                            }
                        },
                        "Publish Docker": {
                            withDockerRegistry([credentialsId: 'docker-registry-creds', url: "https://${DOCKER_REGISTRY}"]) {
                                sh '''
                                    echo "=== Docker Image Publishing ==="
                                    
                                    # 이미지 푸시
                                    docker push ${DOCKER_REGISTRY}/myapp:${CALCULATED_VERSION}
                                    docker push ${DOCKER_REGISTRY}/myapp:latest
                                    
                                    echo "✅ Docker image published"
                                '''
                            }
                        },
                        "Publish to Artifactory": {
                            sh '''
                                echo "=== Artifactory Publishing ==="
                                
                                # 아티팩트를 Artifactory에 업로드
                                curl -u ${ARTIFACTORY_USER}:${ARTIFACTORY_PASSWORD} \
                                    -X PUT "${ARTIFACTORY_URL}/artifactory/libs-release-local/com/mycompany/myapp/${CALCULATED_VERSION}/" \
                                    -T static-assets-${CALCULATED_VERSION}.tar.gz
                                
                                echo "✅ Artifacts published to Artifactory"
                            '''
                        }
                    )
                }
            }
        }
        
        stage('Release Tagging') {
            when {
                branch 'main'
            }
            steps {
                sshagent(['git-ssh-key']) {
                    sh '''
                        echo "=== Release Tagging ==="
                        
                        # Git 태그 생성
                        git tag -a "v${CALCULATED_VERSION}" -m "Release version ${CALCULATED_VERSION}"
                        git push origin "v${CALCULATED_VERSION}"
                        
                        # GitHub Release 생성 (선택적)
                        if [ -n "${GITHUB_TOKEN}" ]; then
                            gh release create "v${CALCULATED_VERSION}" \
                                --title "Release ${CALCULATED_VERSION}" \
                                --notes "Automated release for version ${CALCULATED_VERSION}" \
                                static-assets-${CALCULATED_VERSION}.tar.gz
                        fi
                        
                        echo "✅ Release tagged: v${CALCULATED_VERSION}"
                    '''
                }
            }
        }
    }
    
    post {
        success {
            // 성공 시 아티팩트 메타데이터 기록
            sh '''
                cat > release-metadata.json << EOF
                {
                  "version": "${CALCULATED_VERSION}",
                  "buildNumber": "${BUILD_NUMBER}",
                  "gitCommit": "${GIT_COMMIT}",
                  "branch": "${BRANCH_NAME}",
                  "releaseDate": "$(date -Iseconds)",
                  "artifacts": {
                    "npm": "${NPM_REGISTRY}/myapp/-/myapp-${CALCULATED_VERSION}.tgz",
                    "docker": "${DOCKER_REGISTRY}/myapp:${CALCULATED_VERSION}",
                    "staticAssets": "${ARTIFACTORY_URL}/artifactory/libs-release-local/com/mycompany/myapp/${CALCULATED_VERSION}/static-assets-${CALCULATED_VERSION}.tar.gz"
                  },
                  "qualityGates": {
                    "testsRun": true,
                    "securityScanned": true,
                    "qualityValidated": true
                  }
                }
                EOF
            '''
            archiveArtifacts artifacts: 'release-metadata.json'
        }
    }
}

### 3.2 엔터프라이즈급 아티팩트 저장소 전략

**Nexus Repository Manager 통합**:

```groovy
pipeline {
    agent any
    
    environment {
        NEXUS_URL = 'https://nexus.company.com'
        NEXUS_REPOSITORY = 'maven-releases'
        NEXUS_DOCKER_REPO = 'docker-hosted'
    }
    
    stages {
        stage('Enterprise Artifact Management') {
            steps {
                sh '''
                    echo "=== Enterprise Artifact Repository Strategy ==="
                    
                    # 아티팩트 메타데이터 생성
                    cat > artifact-bom.json << EOF
                    {
                      "artifacts": [
                        {
                          "type": "application",
                          "format": "jar",
                          "coordinates": "com.company:myapp:${CALCULATED_VERSION}",
                          "checksums": {
                            "sha1": "$(sha1sum target/myapp-${CALCULATED_VERSION}.jar | cut -d' ' -f1)",
                            "md5": "$(md5sum target/myapp-${CALCULATED_VERSION}.jar | cut -d' ' -f1)"
                          }
                        },
                        {
                          "type": "container",
                          "format": "docker",
                          "coordinates": "${DOCKER_REGISTRY}/myapp:${CALCULATED_VERSION}",
                          "digest": "$(docker inspect ${DOCKER_REGISTRY}/myapp:${CALCULATED_VERSION} --format='{{.Id}}')"
                        }
                      ],
                      "dependencies": [
                        $(npm list --json | jq '.dependencies | to_entries | map({name: .key, version: .value.version}) | map("{\\"name\\": \\"\(.name)\\", \\"version\\": \\"\(.version)\\"}") | join(",")')
                      ]
                    }
                    EOF
                    
                    echo "✅ Artifact BOM generated"
                '''
                
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'https',
                    nexusUrl: "${NEXUS_URL}",
                    groupId: 'com.company',
                    version: "${CALCULATED_VERSION}",
                    repository: "${NEXUS_REPOSITORY}",
                    credentialsId: 'nexus-credentials',
                    artifacts: [
                        [artifactId: 'myapp', classifier: '', file: 'target/myapp-' + "${CALCULATED_VERSION}" + '.jar', type: 'jar'],
                        [artifactId: 'myapp', classifier: 'sources', file: 'target/myapp-' + "${CALCULATED_VERSION}" + '-sources.jar', type: 'jar'],
                        [artifactId: 'myapp', classifier: 'bom', file: 'artifact-bom.json', type: 'json']
                    ]
                )
            }
        }
    }
}
```

## 4. 알림과 피드백 루프

### 4.1 지능형 알림 시스템

**상황별 맞춤 알림 전략**:

```groovy
pipeline {
    agent any
    
    environment {
        SLACK_CHANNEL = '#ci-notifications'
        TEAMS_WEBHOOK = credentials('teams-webhook-url')
        EMAIL_RECIPIENTS = 'dev-team@company.com'
    }
    
    stages {
        stage('Build Notification Setup') {
            steps {
                script {
                    // 빌드 시작 알림
                    sendBuildStartNotification()
                }
            }
        }
    }
    
    post {
        always {
            script {
                sendDetailedBuildReport()
            }
        }
        failure {
            script {
                sendFailureAnalysis()
            }
        }
        unstable {
            script {
                sendStabilityWarning()
            }
        }
        fixed {
            script {
                sendFixedCelebration()
            }
        }
        regression {
            script {
                sendRegressionAlert()
            }
        }
    }
}

def sendBuildStartNotification() {
    def message = """
    🚀 *Build Started*
    
    *Project:* ${env.JOB_NAME}
    *Build:* #${env.BUILD_NUMBER}
    *Branch:* ${env.BRANCH_NAME}
    *Commit:* ${env.GIT_COMMIT[0..7]}
    *Triggered by:* ${env.BUILD_USER ?: 'SCM Change'}
    
    📊 *Estimated Duration:* ${getEstimatedDuration()} minutes
    🔗 *Build URL:* ${env.BUILD_URL}
    """
    
    slackSend channel: env.SLACK_CHANNEL,
              color: 'warning',
              message: message
}

def sendDetailedBuildReport() {
    def testResults = getTestResults()
    def qualityMetrics = getQualityMetrics()
    def buildDuration = currentBuild.durationString.replace(' and counting', '')
    
    def status = currentBuild.currentResult
    def color = status == 'SUCCESS' ? 'good' : status == 'FAILURE' ? 'danger' : 'warning'
    def emoji = status == 'SUCCESS' ? '✅' : status == 'FAILURE' ? '❌' : '⚠️'
    
    def message = """
    ${emoji} *Build ${status}*
    
    *Project:* ${env.JOB_NAME}
    *Build:* #${env.BUILD_NUMBER} (${buildDuration})
    *Branch:* ${env.BRANCH_NAME}
    
    📊 *Test Results:*
    • Unit Tests: ${testResults.unit.passed}/${testResults.unit.total} passed
    • Integration Tests: ${testResults.integration.passed}/${testResults.integration.total} passed
    • Coverage: ${testResults.coverage}%
    
    📈 *Quality Metrics:*
    • Code Quality: ${qualityMetrics.grade}
    • Complexity: ${qualityMetrics.complexity}
    • Duplication: ${qualityMetrics.duplication}%
    
    🔗 *Links:*
    • <${env.BUILD_URL}|Build Details>
    • <${env.BUILD_URL}testReport/|Test Results>
    • <${env.BUILD_URL}artifact/|Artifacts>
    """
    
    slackSend channel: env.SLACK_CHANNEL,
              color: color,
              message: message
}

def sendFailureAnalysis() {
    def failureStage = getFailedStage()
    def failureReason = getFailureReason()
    def suggestedFix = getSuggestedFix(failureStage, failureReason)
    
    def message = """
    🔥 *Build Failed - Immediate Action Required*
    
    *Project:* ${env.JOB_NAME}
    *Build:* #${env.BUILD_NUMBER}
    *Failed Stage:* ${failureStage}
    *Branch:* ${env.BRANCH_NAME}
    
    💥 *Failure Analysis:*
    \`\`\`
    ${failureReason}
    \`\`\`
    
    🔧 *Suggested Fix:*
    ${suggestedFix}
    
    📱 *Assignee:* <@${getResponsibleDeveloper()}>
    🔗 *Debug:* ${env.BUILD_URL}console
    
    ⏰ *SLA Warning:* Fix required within 2 hours to meet deployment schedule
    """
    
    // 중요한 실패는 여러 채널에 알림
    slackSend channel: '#critical-alerts',
              color: 'danger',
              message: message
              
    slackSend channel: env.SLACK_CHANNEL,
              color: 'danger', 
              message: message
    
    // 이메일 알림도 함께
    emailext(
        subject: "🚨 URGENT: Build Failed - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: message.replace('*', '').replace('`', ''),
        to: env.EMAIL_RECIPIENTS
    )
}

def sendFixedCelebration() {
    def message = """
    🎉 *Build Fixed!* 
    
    *Project:* ${env.JOB_NAME}
    *Build:* #${env.BUILD_NUMBER}
    *Fixed by:* <@${env.BUILD_USER}>
    
    The build is back to green! 🟢
    Great work fixing the issues! 👏
    
    🔗 *Success Details:* ${env.BUILD_URL}
    """
    
    slackSend channel: env.SLACK_CHANNEL,
              color: 'good',
              message: message
}

def getEstimatedDuration() {
    // 최근 성공한 빌드들의 평균 시간 계산
    return "~5" // 실제로는 Jenkins API를 통해 계산
}

def getTestResults() {
    // 실제로는 테스트 결과 파일 파싱
    return [
        unit: [passed: 245, total: 250],
        integration: [passed: 38, total: 40],
        coverage: 87
    ]
}

def getQualityMetrics() {
    // 실제로는 SonarQube API 호출
    return [
        grade: 'A',
        complexity: 8.2,
        duplication: 3.1
    ]
}

def getFailedStage() {
    // 실제로는 빌드 로그 분석
    return "Integration Tests"
}

def getFailureReason() {
    // 실제로는 로그 분석 및 패턴 매칭
    return "Database connection timeout during integration tests. Possibly related to network issues or database overload."
}

def getSuggestedFix(stage, reason) {
    // 실패 패턴에 따른 자동화된 해결책 제안
    def fixes = [
        "Integration Tests": [
            "database": "1. Check database connection pool settings\n2. Verify test data cleanup\n3. Consider increasing timeout values",
            "network": "1. Check network connectivity\n2. Verify firewall settings\n3. Consider using test doubles"
        ]
    ]
    
    return fixes[stage]?.database ?: "Check logs and contact DevOps team"
}

def getResponsibleDeveloper() {
    // Git blame 기반 책임자 식별
    return sh(script: "git log -1 --pretty=format:'%ae' | cut -d'@' -f1", returnStdout: true).trim()
}
```

### 4.2 메트릭 기반 지속 개선

**CI 성능 대시보드와 트렌드 분석**:

```groovy
pipeline {
    agent any
    
    stages {
        stage('CI Metrics Collection') {
            steps {
                sh '''
                    echo "=== CI Metrics Collection ==="
                    
                    # 빌드 시간 메트릭
                    BUILD_START_TIME=$(date +%s)
                    echo $BUILD_START_TIME > build_start_timestamp
                    
                    # 리소스 사용량 모니터링
                    (
                        while true; do
                            TIMESTAMP=$(date +%s)
                            CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | sed 's/%us,//')
                            MEMORY_USAGE=$(free | grep Mem | awk '{printf "%.2f", $3/$2 * 100.0}')
                            DISK_IO=$(iostat -d 1 1 | grep -E "sda|nvme" | awk '{sum+=$4} END {print sum}' || echo "0")
                            
                            echo "$TIMESTAMP,$CPU_USAGE,$MEMORY_USAGE,$DISK_IO" >> resource_metrics.csv
                            sleep 10
                        done
                    ) &
                    MONITOR_PID=$!
                    echo $MONITOR_PID > monitor.pid
                    
                    # 테스트 실행 시간 개별 측정
                    echo "Measuring individual test execution times..."
                    npm run test:unit -- --reporter=json > unit-test-detailed.json
                '''
            }
        }
    }
    
    post {
        always {
            sh '''
                # 모니터링 정지
                if [ -f monitor.pid ]; then
                    kill $(cat monitor.pid) 2>/dev/null || true
                fi
                
                # 빌드 완료 시간 기록
                BUILD_END_TIME=$(date +%s)
                BUILD_START_TIME=$(cat build_start_timestamp)
                BUILD_DURATION=$((BUILD_END_TIME - BUILD_START_TIME))
                
                # CI 메트릭 집계
                cat > ci-metrics.json << EOF
                {
                  "buildNumber": "${BUILD_NUMBER}",
                  "timestamp": "$(date -Iseconds)",
                  "duration": {
                    "total": $BUILD_DURATION,
                    "stages": {
                      "checkout": $(grep -o "Checkout.*took [0-9]*" ${BUILD_URL}consoleText | awk '{print $NF}' || echo 0),
                      "test": $(grep -o "test.*took [0-9]*" ${BUILD_URL}consoleText | awk '{print $NF}' || echo 0),
                      "build": $(grep -o "build.*took [0-9]*" ${BUILD_URL}consoleText | awk '{print $NF}' || echo 0)
                    }
                  },
                  "resources": {
                    "maxCpuUsage": $(awk -F',' 'BEGIN{max=0} {if($2>max) max=$2} END{print max}' resource_metrics.csv || echo 0),
                    "maxMemoryUsage": $(awk -F',' 'BEGIN{max=0} {if($3>max) max=$3} END{print max}' resource_metrics.csv || echo 0),
                    "averageDiskIO": $(awk -F',' 'BEGIN{sum=0; count=0} {sum+=$4; count++} END{if(count>0) print sum/count; else print 0}' resource_metrics.csv || echo 0)
                  },
                  "testMetrics": {
                    "totalTests": $(cat unit-test-detailed.json | jq '.stats.tests // 0'),
                    "passedTests": $(cat unit-test-detailed.json | jq '.stats.passes // 0'),
                    "failedTests": $(cat unit-test-detailed.json | jq '.stats.failures // 0'),
                    "slowestTest": $(cat unit-test-detailed.json | jq '.tests | max_by(.duration) | .title + " (" + (.duration|tostring) + "ms)"' 2>/dev/null || echo "null"),
                    "averageTestTime": $(cat unit-test-detailed.json | jq '[.tests[].duration] | add / length' 2>/dev/null || echo 0)
                  },
                  "artifacts": {
                    "count": $(ls -1 target/*.jar 2>/dev/null | wc -l || echo 0),
                    "totalSize": "$(du -sh target/ 2>/dev/null | cut -f1 || echo '0B')"
                  }
                }
                EOF
                
                # InfluxDB로 메트릭 전송 (선택적)
                if [ -n "$INFLUXDB_URL" ]; then
                    curl -X POST "$INFLUXDB_URL/write?db=jenkins_metrics" \
                        --data-binary "
                        jenkins_build,job=${JOB_NAME},build_number=${BUILD_NUMBER} duration=${BUILD_DURATION}i,result=\"${BUILD_RESULT}\" $(date +%s)000000000
                        jenkins_tests,job=${JOB_NAME} total=$(cat unit-test-detailed.json | jq '.stats.tests // 0')i,passed=$(cat unit-test-detailed.json | jq '.stats.passes // 0')i $(date +%s)000000000
                        "
                fi
                
                echo "✅ CI metrics collected and sent"
            '''
            
            archiveArtifacts artifacts: 'ci-metrics.json,resource_metrics.csv', fingerprint: true
        }
    }
}
```

## 5. 실전 문제 해결과 트러블슈팅

### 5.1 일반적인 CI 문제와 해결 방안

**빈번한 CI 실패 패턴 분석**:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Pre-flight Health Checks') {
            steps {
                sh '''
                    echo "=== CI Environment Health Check ==="
                    
                    # 1. 시스템 리소스 확인
                    echo "System Resources:"
                    echo "- Available Memory: $(free -h | grep ^Mem | awk '{print $7}')"
                    echo "- Available Disk: $(df -h / | tail -1 | awk '{print $4}')"
                    echo "- CPU Load: $(uptime | awk '{print $NF}')"
                    
                    # 2. 의존성 서비스 상태 확인
                    echo "Dependency Health:"
                    
                    # Database connectivity
                    if ! nc -z database-host 5432; then
                        echo "❌ Database connection failed"
                        echo "Suggested fix: Check database server status and network connectivity"
                        exit 1
                    fi
                    echo "✅ Database connection OK"
                    
                    # Redis connectivity  
                    if ! nc -z redis-host 6379; then
                        echo "⚠️ Redis connection failed - some tests may be skipped"
                    else
                        echo "✅ Redis connection OK"
                    fi
                    
                    # External API dependencies
                    for API in https://api.service1.com/health https://api.service2.com/health; do
                        if curl -f --connect-timeout 10 $API > /dev/null 2>&1; then
                            echo "✅ $API is available"
                        else
                            echo "⚠️ $API is not responding - some integration tests may fail"
                        fi
                    done
                    
                    # 3. 테스트 데이터 상태 확인
                    echo "Test Data Validation:"
                    
                    # 테스트 데이터베이스 스키마 버전 확인
                    SCHEMA_VERSION=$(psql $TEST_DATABASE_URL -t -c "SELECT version FROM schema_migrations ORDER BY version DESC LIMIT 1;" 2>/dev/null || echo "unknown")
                    echo "Test DB Schema Version: $SCHEMA_VERSION"
                    
                    # 테스트 파일 무결성 확인
                    if [ -d "test/fixtures" ]; then
                        FIXTURE_COUNT=$(find test/fixtures -name "*.json" | wc -l)
                        echo "Test fixtures available: $FIXTURE_COUNT files"
                    fi
                    
                    echo "✅ Pre-flight checks completed"
                '''
            }
        }
        
        stage('Flaky Test Detection') {
            steps {
                sh '''
                    echo "=== Flaky Test Detection ==="
                    
                    # 과거 빌드에서 불안정한 테스트 패턴 분석
                    if [ -f "test-history.json" ]; then
                        node << 'EOF'
                        const testHistory = require('./test-history.json');
                        const flakyThreshold = 0.8; // 80% 미만 성공률
                        
                        const testStats = {};
                        
                        // 테스트 성공률 계산
                        testHistory.builds.forEach(build => {
                            build.tests.forEach(test => {
                                if (!testStats[test.name]) {
                                    testStats[test.name] = { total: 0, passed: 0 };
                                }
                                testStats[test.name].total++;
                                if (test.status === 'passed') {
                                    testStats[test.name].passed++;
                                }
                            });
                        });
                        
                        // Flaky 테스트 식별
                        const flakyTests = Object.entries(testStats)
                            .map(([name, stats]) => ({
                                name,
                                successRate: stats.passed / stats.total,
                                total: stats.total
                            }))
                            .filter(test => test.successRate < flakyThreshold && test.total >= 5)
                            .sort((a, b) => a.successRate - b.successRate);
                        
                        if (flakyTests.length > 0) {
                            console.log('🔄 Flaky tests detected:');
                            flakyTests.forEach(test => {
                                console.log(`- ${test.name}: ${(test.successRate * 100).toFixed(1)}% success rate`);
                            });
                            
                            // Flaky 테스트 재실행 전략
                            console.log('\\nApplying flaky test mitigation...');
                        } else {
                            console.log('✅ No flaky tests detected');
                        }
                        EOF
                    fi
                '''
            }
        }
        
        stage('Intelligent Test Retry') {
            steps {
                sh '''
                    echo "=== Intelligent Test Retry Strategy ==="
                    
                    # 첫 번째 테스트 실행
                    if npm run test:all -- --reporter=json > test-results-1.json 2>&1; then
                        echo "✅ All tests passed on first attempt"
                        cp test-results-1.json test-results-final.json
                    else
                        echo "⚠️ Some tests failed, analyzing failures..."
                        
                        # 실패한 테스트 분석
                        node << 'EOF'
                        try {
                            const results = require('./test-results-1.json');
                            const failedTests = results.tests.filter(test => test.err);
                            
                            console.log(`Failed tests: ${failedTests.length}`);
                            
                            // 재시도 가능한 실패 패턴 확인
                            const retryableFailures = failedTests.filter(test => {
                                const errorMsg = test.err.message.toLowerCase();
                                return errorMsg.includes('timeout') || 
                                       errorMsg.includes('connection') ||
                                       errorMsg.includes('network') ||
                                       errorMsg.includes('flaky');
                            });
                            
                            if (retryableFailures.length > 0) {
                                console.log(`Retryable failures: ${retryableFailures.length}`);
                                
                                // 재시도할 테스트 목록 생성
                                const retryList = retryableFailures.map(test => test.fullTitle).join('|');
                                require('fs').writeFileSync('retry-tests.txt', retryList);
                            }
                        } catch (e) {
                            console.log('Could not parse test results for retry analysis');
                        }
                        EOF
                        
                        # 선별적 재시도
                        if [ -f "retry-tests.txt" ]; then
                            echo "🔄 Retrying flaky tests..."
                            RETRY_PATTERN=$(cat retry-tests.txt)
                            npm run test:all -- --grep "$RETRY_PATTERN" --reporter=json > test-results-retry.json
                            
                            # 결과 병합
                            node << 'EOF'
                            const original = require('./test-results-1.json');
                            const retry = require('./test-results-retry.json');
                            
                            // 재시도 성공한 테스트로 원본 결과 업데이트
                            retry.tests.forEach(retryTest => {
                                if (!retryTest.err) {
                                    const originalIndex = original.tests.findIndex(
                                        test => test.fullTitle === retryTest.fullTitle
                                    );
                                    if (originalIndex !== -1) {
                                        original.tests[originalIndex] = retryTest;
                                    }
                                }
                            });
                            
                            require('fs').writeFileSync('test-results-final.json', JSON.stringify(original, null, 2));
                            EOF
                        else
                            cp test-results-1.json test-results-final.json
                        fi
                        
                        # 최종 결과 검증
                        FINAL_FAILURES=$(cat test-results-final.json | jq '.stats.failures')
                        if [ "$FINAL_FAILURES" -gt 0 ]; then
                            echo "❌ Tests still failing after retry"
                            exit 1
                        else
                            echo "✅ All tests passed after intelligent retry"
                        fi
                    fi
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'test-results-*.json', allowEmptyArchive: true
                }
            }
        }
    }
}
```

### 5.2 성능 병목 식별과 최적화

**CI 파이프라인 성능 프로파일링**:

```groovy
pipeline {
    agent any
    
    environment {
        PROFILING_ENABLED = 'true'
    }
    
    stages {
        stage('Performance Profiling Setup') {
            when {
                environment name: 'PROFILING_ENABLED', value: 'true'
            }
            steps {
                sh '''
                    echo "=== CI Performance Profiling Setup ==="
                    
                    # 성능 프로파일링 도구 설치
                    npm install -g clinic
                    
                    # 베이스라인 성능 측정 시작
                    echo "$(date +%s)" > profile_start_time
                    
                    # 시스템 리소스 베이스라인
                    echo "Initial system state:" > performance_baseline.txt
                    echo "CPU: $(nproc) cores" >> performance_baseline.txt
                    echo "Memory: $(free -h | grep ^Mem | awk '{print $2}')" >> performance_baseline.txt
                    echo "Disk: $(df -h / | tail -1 | awk '{print $2}')" >> performance_baseline.txt
                '''
            }
        }
        
        stage('Dependency Installation Profiling') {
            steps {
                sh '''
                    echo "=== Dependency Installation Performance ==="
                    
                    # npm install 성능 측정
                    START_TIME=$(date +%s.%N)
                    
                    if [ "$PROFILING_ENABLED" = "true" ]; then
                        clinic doctor --on-port 'npm run build:profile' -- npm install
                    else
                        npm install
                    fi
                    
                    END_TIME=$(date +%s.%N)
                    INSTALL_DURATION=$(echo "$END_TIME - $START_TIME" | bc)
                    
                    echo "Dependency installation took: ${INSTALL_DURATION}s"
                    echo "install_duration=$INSTALL_DURATION" >> build_metrics.env
                    
                    # 캐시 효율성 분석
                    NPM_CACHE_SIZE=$(du -sh ~/.npm 2>/dev/null | cut -f1 || echo "0B")
                    echo "NPM cache size: $NPM_CACHE_SIZE"
                    echo "npm_cache_size=$NPM_CACHE_SIZE" >> build_metrics.env
                '''
            }
        }
        
        stage('Test Execution Profiling') {
            steps {
                sh '''
                    echo "=== Test Execution Performance Analysis ==="
                    
                    # 테스트 실행 성능 프로파일링
                    if [ "$PROFILING_ENABLED" = "true" ]; then
                        # CPU 프로파일링
                        clinic flame -- npm run test:unit
                        
                        # 메모리 프로파일링  
                        clinic bubbleprof -- npm run test:unit
                        
                        # 이벤트 루프 지연 프로파일링
                        clinic doctor -- npm run test:unit
                    else
                        npm run test:unit
                    fi
                    
                    # 테스트별 실행 시간 분석
                    npm run test:unit -- --reporter=json > test_performance.json
                    
                    # 느린 테스트 식별
                    node << 'EOF'
                    const testResults = require('./test_performance.json');
                    const slowTests = testResults.tests
                        .filter(test => test.duration > 1000) // 1초 이상
                        .sort((a, b) => b.duration - a.duration)
                        .slice(0, 10);
                    
                    if (slowTests.length > 0) {
                        console.log('🐌 Slowest tests:');
                        slowTests.forEach((test, index) => {
                            console.log(`${index + 1}. ${test.title}: ${test.duration}ms`);
                        });
                        
                        // 최적화 제안
                        console.log('\\n💡 Optimization suggestions:');
                        console.log('- Consider parallelizing slow tests');
                        console.log('- Review test setup/teardown efficiency');
                        console.log('- Use test doubles for external dependencies');
                    }
                    EOF
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: '.clinic/**/*', allowEmptyArchive: true
                }
            }
        }
        
        stage('Build Process Optimization') {
            steps {
                sh '''
                    echo "=== Build Process Performance Analysis ==="
                    
                    # 빌드 단계별 시간 측정
                    BUILD_START=$(date +%s.%N)
                    
                    # Webpack 빌드 분석
                    npm run build -- --profile --json > webpack-stats.json
                    
                    BUILD_END=$(date +%s.%N)
                    BUILD_DURATION=$(echo "$BUILD_END - $BUILD_START" | bc)
                    
                    echo "Build took: ${BUILD_DURATION}s"
                    echo "build_duration=$BUILD_DURATION" >> build_metrics.env
                    
                    # Webpack 번들 분석
                    npx webpack-bundle-analyzer webpack-stats.json --mode json --report bundle-analysis.json
                    
                    # 빌드 최적화 제안 생성
                    node << 'EOF'
                    const bundleStats = require('./webpack-stats.json');
                    const analysis = require('./bundle-analysis.json');
                    
                    console.log('📊 Bundle Analysis:');
                    console.log(`Total bundle size: ${(bundleStats.assets.reduce((sum, asset) => sum + asset.size, 0) / 1024 / 1024).toFixed(2)}MB`);
                    
                    // 큰 의존성 식별
                    const largeDeps = bundleStats.modules
                        .filter(mod => mod.size > 100000) // 100KB 이상
                        .sort((a, b) => b.size - a.size)
                        .slice(0, 5);
                    
                    if (largeDeps.length > 0) {
                        console.log('\\n🔍 Large dependencies:');
                        largeDeps.forEach(dep => {
                            console.log(`- ${dep.name}: ${(dep.size / 1024).toFixed(2)}KB`);
                        });
                    }
                    EOF
                '''
            }
        }
        
        stage('Performance Report Generation') {
            steps {
                sh '''
                    echo "=== Performance Report Generation ==="
                    
                    # 성능 메트릭 수집
                    PROFILE_END_TIME=$(date +%s)
                    PROFILE_START_TIME=$(cat profile_start_time)
                    TOTAL_DURATION=$((PROFILE_END_TIME - PROFILE_START_TIME))
                    
                    # 성능 리포트 생성
                    cat > performance_report.json << EOF
                    {
                      "buildNumber": "${BUILD_NUMBER}",
                      "timestamp": "$(date -Iseconds)",
                      "performance": {
                        "totalDuration": $TOTAL_DURATION,
                        "installDuration": $(grep install_duration build_metrics.env | cut -d= -f2),
                        "buildDuration": $(grep build_duration build_metrics.env | cut -d= -f2),
                        "npmCacheSize": "$(grep npm_cache_size build_metrics.env | cut -d= -f2)"
                      },
                      "recommendations": [
                        {
                          "category": "dependency_management",
                          "suggestion": "Consider using npm ci instead of npm install for faster, reproducible builds"
                        },
                        {
                          "category": "caching",
                          "suggestion": "Implement Docker layer caching for dependency installation"
                        },
                        {
                          "category": "parallelization", 
                          "suggestion": "Split tests into parallel jobs for faster feedback"
                        }
                      ]
                    }
                    EOF
                    
                    echo "Performance report generated:"
                    cat performance_report.json | jq .
                '''
                
                archiveArtifacts artifacts: 'performance_report.json,webpack-stats.json,bundle-analysis.json'
            }
        }
    }
}
```

## 6. 실습 과제 - 난이도별 학습

### 6.1 기초 실습 (입문자용)

**실습 1: 기본 테스트 자동화 파이프라인**

```groovy
// 여러분이 작성해야 할 Jenkinsfile
pipeline {
    agent any
    
    stages {
        stage('Setup') {
            steps {
                // TODO: 의존성 설치
                
                // TODO: 테스트 환경 확인
            }
        }
        
        stage('Unit Tests') {
            steps {
                // TODO: 단위 테스트 실행
                
                // TODO: 테스트 결과를 JUnit 포맷으로 생성
            }
            post {
                always {
                    // TODO: 테스트 결과 게시
                }
            }
        }
        
        stage('Build') {
            steps {
                // TODO: 애플리케이션 빌드
                
                // TODO: 빌드 아티팩트 생성
            }
            post {
                success {
                    // TODO: 아티팩트 아카이브
                }
            }
        }
    }
    
    post {
        // TODO: 빌드 상태별 알림 설정
    }
}
```

**정답 예시**:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Setup') {
            steps {
                echo "Setting up build environment..."
                sh 'npm install'
                
                echo "Verifying test environment:"
                sh 'npm run test:check'
            }
        }
        
        stage('Unit Tests') {
            steps {
                echo "Running unit tests..."
                sh 'npm run test:unit -- --reporter=junit --output-file=unit-test-results.xml'
                
                echo "Checking test coverage..."
                sh 'npm run test:coverage'
            }
            post {
                always {
                    publishTestResults testResultsPattern: 'unit-test-results.xml'
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
        
        stage('Build') {
            steps {
                echo "Building application..."
                sh 'npm run build'
                
                echo "Generating build metadata..."
                sh '''
                    echo "Build Number: ${BUILD_NUMBER}" > build-info.txt
                    echo "Build Date: $(date)" >> build-info.txt
                    echo "Git Commit: ${GIT_COMMIT}" >> build-info.txt
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: 'dist/**, build-info.txt', fingerprint: true
                }
            }
        }
    }
    
    post {
        always {
            echo "Build completed with status: ${currentBuild.currentResult}"
        }
        success {
            echo "✅ Build successful! All tests passed."
        }
        failure {
            echo "❌ Build failed! Check the logs for details."
        }
        unstable {
            echo "⚠️ Build unstable! Some tests may have failed."
        }
    }
}
```

### 6.2 중급 실습 (실무자용)

**실습 2: 품질 게이트가 있는 CI 파이프라인**

```text
다음 요구사항을 만족하는 CI 파이프라인을 구현하세요:

기술적 요구사항:
- 단위 테스트 커버리지 80% 이상
- 통합 테스트 포함 (데이터베이스 연동)
- 정적 코드 분석 (ESLint + SonarQube)
- 보안 취약점 스캔 (npm audit)
- 성능 테스트 (부하 테스트)

품질 게이트:
- 모든 테스트 통과 필수
- 코드 품질 등급 B 이상
- Critical/High 보안 취약점 0개
- 빌드 시간 10분 이내

배포 준비:
- Semantic versioning 자동 적용
- 다중 포맷 아티팩트 생성 (JAR, Docker, TAR)
- 메타데이터와 함께 아티팩트 저장소에 업로드
```

### 6.3 고급 실습 (전문가용)

**실습 3: Zero-Downtime 배포를 위한 포괄적 CI/CD**

```text
다음과 같은 엔터프라이즈급 요구사항을 충족하는 CI 시스템을 설계하고 구현하세요:

복합 요구사항:
- 마이크로서비스 아키텍처 지원 (5개 서비스)
- API 계약 테스트 (Pact/OpenAPI)
- 카나리 배포 지원 (트래픽 점진적 증가)
- 자동 롤백 (SLA 위반 시)
- 멀티 클라우드 배포 (AWS + Azure)

운영 요구사항:
- 99.9% 가용성 SLA 보장
- 15분 이내 배포 완료
- 실시간 모니터링과 알림
- 컴플라이언스 로깅 (SOX, GDPR)
- 재해 복구 시나리오 지원

성능 요구사항:
- 빌드 시간 최적화 (병렬 처리)
- 테스트 실행 시간 최소화
- 아티팩트 크기 최적화
- 네트워크 대역폭 효율성
```

---

## ✅ 종합 평가 퀴즈

### 객관식 문제

1. **Testing Pyramid에서 가장 많은 비중을 차지해야 하는 테스트 유형은?**
   - A) End-to-End 테스트
   - B) 통합 테스트
   - C) 단위 테스트
   - D) 성능 테스트

2. **CI 파이프라인에서 품질 게이트(Quality Gate)의 주요 목적은?**
   - A) 빌드 시간 단축
   - B) 배포 불가능한 코드 조기 차단
   - C) 개발자 생산성 향상
   - D) 인프라 비용 절감

3. **Semantic Versioning에서 MAJOR 버전을 증가시키는 경우는?**
   - A) 새로운 기능 추가
   - B) 버그 수정
   - C) 하위 호환성을 깨는 변경
   - D) 문서 업데이트

### 서술형 문제

1. **Flaky 테스트가 CI/CD 파이프라인에 미치는 영향과 이를 해결하기 위한 3가지 전략을 설명하세요.**

2. **대규모 마이크로서비스 환경에서 아티팩트 관리의 핵심 과제 5가지와 각각의 해결 방안을 제시하세요.**

3. **CI 파이프라인의 성능 최적화를 위한 병목 지점 식별 방법과 구체적인 개선 기법들을 설명하세요.**

---

## 🎯 학습 목표 달성도 체크

이번 6주차를 통해 다음 역량을 달성했는지 확인해보세요:

### ✅ 기술적 역량
- [ ] 다층 테스트 전략 설계와 구현
- [ ] SonarQube를 활용한 코드 품질 게이트 구축
- [ ] 동적 품질 기준 설정과 적응형 검증
- [ ] Semantic versioning과 자동 태깅 시스템
- [ ] 엔터프라이즈급 아티팩트 관리 전략
- [ ] CI 성능 프로파일링과 최적화

### ✅ 운영 역량
- [ ] Flaky 테스트 탐지와 지능형 재시도
- [ ] 상황별 맞춤 알림 시스템 구축
- [ ] CI 메트릭 수집과 트렌드 분석
- [ ] 성능 병목 식별과 해결
- [ ] Zero-defect 배포를 위한 품질 보장

---

**다음 주 예고**: DevSecOps Mastery 7주차에서는 **SAST(Static Application Security Testing)**를 다룹니다. 소스 코드를 정적 분석하여 보안 취약점을 사전에 탐지하고, SecDevOps 파이프라인에 보안을 네이티브하게 통합하는 방법을 학습합니다.

**Test Early, Test Often, Test Everywhere.**

// 헬퍼 함수들
def getMinorVersion() {
    // feature 브랜치에서는 minor 버전 증가
    if (env.BRANCH_NAME.startsWith('feature/')) {
        return sh(script: "git tag | grep -E '^v[0-9]+\\.' | sort -V | tail -1 | cut -d. -f2", returnStdout: true).trim() + 1
    }
    return sh(script: "git tag | grep -E '^v[0-9]+\\.' | sort -V | tail -1 | cut -d. -f2", returnStdout: true).trim()
}

def determineVersionBump() {
    def commitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
    
    if (commitMessage.contains('BREAKING CHANGE:')) {
        return 'major'
    } else if (commitMessage.startsWith('feat:')) {
        return 'minor'
    } else {
        return 'patch'
    }
}

def calculateNewVersion(versionBump) {
    def currentVersion = sh(script: 'git tag | grep -E "^v[0-9]+" | sort -V | tail -1 | sed "s/v//"', returnStdout: true).trim()
    if (!currentVersion) {
        return "1.0.0"
    }
    
    def (major, minor, patch) = currentVersion.split('\\.')
    
    switch(versionBump) {
        case 'major':
            return "${major.toInteger() + 1}.0.0"
        case 'minor':
            return "${major}.${minor.toInteger() + 1}.0"
        case 'patch':
            return "${major}.${minor}.${patch.toInteger() + 1}"
    }
}
```
