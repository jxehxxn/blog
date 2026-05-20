---
layout: post
title: "DevSecOps Mastery: 5주차 - 함께 쓰는 Docker와 Jenkins"
date: 2026-05-20 14:20:00 +0900
---

# DevSecOps Mastery: 5주차 - 함께 쓰는 Docker와 Jenkins

## 서론

지난 4주 동안 파이프라인의 기본 뼈대를 다뤘다. 그런데 실제 운영에 들어가면 또 다른 문제가 기다린다. 바로 의존성 충돌이다.

### 현실에서 벌어지는 문제

실제 기업의 풍경부터 짚자.

#### Netflix의 2019년 사례

Netflix는 2,500개의 마이크로서비스를 굴리는데, 서비스마다 런타임 요구사항이 다릅니다.
- 서비스 A: Node.js 12 + Python 3.7
- 서비스 B: Node.js 16 + Python 3.9  
- 서비스 C: Java 8 + Maven 3.6
- 서비스 D: Java 11 + Gradle 7.0

이걸 전통적인 방식으로 관리하려면, 수백 대의 Jenkins 에이전트마다 도구와 버전을 일일이 맞춰야 합니다. 결과는 뻔합니다. 한 에이전트에 여러 버전을 깔면 충돌이 잦고, 개발 환경과 CI 환경이 미묘하게 어긋나 빌드가 깨집니다. 새 프로젝트가 생길 때마다 에이전트를 다시 손봐야 하고, 오래 유지된 에이전트에는 취약점이 차곡차곡 쌓입니다.

#### Google의 접근법

Google은 이 문제를 풀려고 모든 빌드를 격리된 컨테이너 안에서 돌리는 Hermetic Build 체계를 만들었습니다. 빌드마다 완전히 독립된 환경에서 돌아가니, 빌드끼리 서로 영향을 주지 않습니다.

오늘은 Docker를 Jenkins와 묶어 Google 수준의 빌드 격리와 일관성을 만드는 방법을 다룬다.

## 1. 컨테이너화 CI/CD의 근본 원리

### 1.1 전통적 CI vs 컨테이너화 CI 비교

#### 전통적인 Jenkins 에이전트 방식

```bash
# Jenkins 에이전트 설정 (문제점 많음)
┌─────────────────────────────────────┐
│ Jenkins Agent (Ubuntu 20.04)       │
│ ├── Node.js 12, 14, 16, 18        │ ← 버전 충돌 위험
│ ├── Python 3.7, 3.8, 3.9         │ ← 환경 오염
│ ├── Java 8, 11, 17                │ ← 메모리 낭비
│ ├── Maven 3.6, 3.8                │ ← 관리 복잡성
│ └── 누적된 캐시, 로그, 임시파일      │ ← 예측 불가능
└─────────────────────────────────────┘
```

#### 컨테이너화된 Jenkins 파이프라인

```bash
# 각 빌드마다 격리된 환경
Build A ┌─────────────────┐    Build B ┌─────────────────┐
       │ node:18-alpine  │           │ python:3.9-slim │
       │ ├── Clean env   │           │ ├── Clean env   │
       │ ├── Exact deps  │           │ ├── Exact deps  │
       │ └── Ephemeral   │           │ └── Ephemeral   │
       └─────────────────┘           └─────────────────┘
             ↓ 빌드 완료 시 자동 삭제     ↓ 빌드 완료 시 자동 삭제
```

### 1.2 기업 환경에서의 이점

#### 1) 재현성 (Reproducibility)

```groovy
// 정확히 동일한 환경 보장
pipeline {
    agent {
        docker {
            // 이 이미지의 SHA256 해시는 항상 동일한 환경을 보장
            image 'node:18.17.1-alpine3.18@sha256:f77a1aef2da8d83e45ec990f45df50f1a286c5fe8bbfb8c6e4246c6389705c0b'
        }
    }
    stages {
        stage('Reproducible Build') {
            steps {
                sh '''
                    # 정확한 버전 출력 - 언제나 동일
                    node --version    # v18.17.1
                    npm --version     # 9.8.1
                    alpine-version    # 3.18.2
                    
                    # 의존성 설치 - 정확한 버전들
                    npm ci  # package-lock.json 기준 정확한 버전
                '''
            }
        }
    }
}
```

#### 2) 보안 격리 (Security Isolation)

```groovy
pipeline {
    agent {
        docker {
            image 'node:18-alpine'
            // 보안 강화 옵션들
            args '''
                --user 1000:1000
                --read-only
                --tmpfs /tmp
                --tmpfs /var/tmp
                --tmpfs /home/node
                --security-opt no-new-privileges
                --cap-drop ALL
                --cap-add CHOWN
                --cap-add SETGID
                --cap-add SETUID
            '''
        }
    }
    stages {
        stage('Secure Build') {
            steps {
                sh '''
                    # 컨테이너는 읽기 전용 파일시스템에서 실행
                    # 악성 코드가 파일시스템을 조작할 수 없음
                    npm ci
                    npm run build
                    npm test
                '''
            }
        }
    }
}
```

#### 3) 리소스 효율성 (Resource Efficiency)

```groovy
pipeline {
    agent {
        docker {
            image 'node:18-alpine'  // 크기: ~175MB
            // 리소스 제한
            args '''
                --memory=2g
                --memory-swap=2g
                --cpu-quota=100000
                --cpu-period=100000
            '''
        }
    }
    
    environment {
        // Node.js 메모리 제한
        NODE_OPTIONS = '--max-old-space-size=1536'
    }
    
    stages {
        stage('Resource Constrained Build') {
            steps {
                sh '''
                    echo "Available memory: $(free -h | grep Mem | awk '{print $7}')"
                    echo "Available CPU: $(nproc)"
                    
                    # 메모리 제한 하에서 빌드 수행
                    npm ci
                    npm run build
                '''
            }
        }
    }
}
```

### 1.3 실제 기업의 컨테이너화 전환 사례

#### Spotify의 2018년 전환

- Before: 200개 Jenkins 에이전트, 각각 다른 구성
- After: 모든 빌드를 Docker 컨테이너로 전환
- 결과: 빌드 일관성은 99.9%까지 올라갔고, 에이전트 관리 시간은 90% 줄었으며, 신규 프로젝트 온보딩은 1주에서 하루로 단축됐다.

```groovy
// Spotify 스타일 표준화된 파이프라인
@Library('spotify-pipeline-library@main') _

dockerPipeline([
    language: 'node',
    version: '18',
    testFramework: 'jest',
    buildTool: 'webpack',
    deployTarget: 'kubernetes'
])
```

#### Shopify의 Ruby 생태계 표준화

```groovy
pipeline {
    agent {
        docker {
            // Shopify 표준 Ruby 스택
            image 'shopify/ruby:3.1-node16-postgres13'
            args '-v bundler-cache:/usr/local/bundle'
        }
    }
    
    stages {
        stage('Shopify Stack Build') {
            parallel {
                stage('Ruby Tests') {
                    steps {
                        sh '''
                            bundle install --jobs=4
                            bundle exec rspec --format progress
                        '''
                    }
                }
                stage('Node Tests') {
                    steps {
                        sh '''
                            npm ci
                            npm run test:unit
                        '''
                    }
                }
                stage('Database Tests') {
                    steps {
                        sh '''
                            bundle exec rake db:create db:schema:load
                            bundle exec rspec spec/models/
                        '''
                    }
                }
            }
        }
    }
}
```

## 2. Docker Agent 고급 설정과 최적화

### 2.1 기본 Docker Agent 구성

#### 단순 Docker Agent

```groovy
pipeline {
    agent {
        docker { 
            image 'node:18-alpine'
            // Jenkins가 현재 워크스페이스를 컨테이너에 자동 마운트
            // 기본 위치: /var/jenkins_home/workspace/{job_name}
        }
    }
    
    stages {
        stage('Environment Check') {
            steps {
                sh '''
                    echo "Container OS: $(cat /etc/os-release | grep PRETTY_NAME)"
                    echo "Working directory: $(pwd)"
                    echo "Node version: $(node --version)"
                    echo "NPM version: $(npm --version)"
                    echo "Available space: $(df -h . | tail -1 | awk '{print $4}')"
                '''
            }
        }
    }
}
```

### 2.2 고급 Docker Agent 설정

#### 캐시와 볼륨 최적화

```groovy
pipeline {
    agent {
        docker {
            image 'node:18-alpine'
            // 고급 설정
            args '''
                -v npm-cache:/root/.npm
                -v node-modules-cache:/app/node_modules
                -e NPM_CONFIG_CACHE=/root/.npm
                -e CI=true
                -e NODE_ENV=production
                --shm-size=1g
            '''
            // 컨테이너 이름 지정 (디버깅 용이)
            alwaysPull false  // 이미지가 있으면 재사용
            reuseNode true   // 같은 노드에서 계속 실행
        }
    }
    
    stages {
        stage('Optimized Build') {
            steps {
                sh '''
                    echo "NPM Cache dir: $NPM_CONFIG_CACHE"
                    echo "Cache contents:"
                    ls -la $NPM_CONFIG_CACHE 2>/dev/null || echo "Empty cache"
                    
                    # 캐시를 활용한 빠른 설치
                    npm ci --prefer-offline --no-audit
                    
                    # 빌드 수행
                    npm run build
                    
                    echo "Build artifacts:"
                    ls -la dist/ 2>/dev/null || echo "No dist directory"
                '''
            }
        }
    }
}
```

### 2.3 다중 컨테이너 파이프라인

#### 서비스 의존성이 있는 복잡한 빌드

```groovy
pipeline {
    agent none
    
    stages {
        stage('Multi-Container Build') {
            parallel {
                stage('Backend Build') {
                    agent {
                        docker {
                            image 'maven:3.8.6-openjdk-17'
                            args '''
                                -v maven-cache:/root/.m2
                                -e MAVEN_OPTS="-Xmx2048m -XX:MaxPermSize=512m"
                            '''
                        }
                    }
                    steps {
                        dir('backend') {
                            sh '''
                                echo "Maven version: $(mvn --version | head -1)"
                                mvn clean package -DskipTests=false -B
                            '''
                        }
                    }
                    post {
                        always {
                            publishTestResults testResultsPattern: 'backend/target/surefire-reports/*.xml'
                        }
                    }
                }
                
                stage('Frontend Build') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            args '-v npm-cache:/root/.npm'
                        }
                    }
                    steps {
                        dir('frontend') {
                            sh '''
                                echo "Node version: $(node --version)"
                                npm ci --prefer-offline
                                npm run lint
                                npm run test -- --coverage --watchAll=false
                                npm run build
                            '''
                        }
                    }
                    post {
                        always {
                            publishTestResults testResultsPattern: 'frontend/test-results.xml'
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'frontend/coverage',
                                reportFiles: 'index.html',
                                reportName: 'Frontend Coverage Report'
                            ])
                        }
                    }
                }
                
                stage('Database Migration Test') {
                    agent {
                        docker {
                            image 'postgres:15-alpine'
                            args '''
                                -e POSTGRES_PASSWORD=testpass
                                -e POSTGRES_DB=testdb
                                --tmpfs /var/lib/postgresql/data
                            '''
                        }
                    }
                    steps {
                        sh '''
                            # PostgreSQL 시작 대기
                            until pg_isready -U postgres; do
                                echo "Waiting for PostgreSQL to start..."
                                sleep 2
                            done
                            
                            echo "PostgreSQL is ready"
                            psql -U postgres -d testdb -c "SELECT version();"
                            
                            # 마이그레이션 스크립트 실행
                            psql -U postgres -d testdb -f backend/src/main/resources/db/migration/init.sql
                        '''
                    }
                }
            }
        }
    }
}
```

### 2.4 동적 Docker 이미지 선택

#### 환경과 브랜치에 따른 동적 이미지 선택

```groovy
pipeline {
    agent none
    
    environment {
        // 브랜치에 따른 이미지 전략
        DOCKER_IMAGE = script {
            switch(env.BRANCH_NAME) {
                case 'main':
                    return 'node:18-alpine'  // Stable LTS
                case 'develop':
                    return 'node:19-alpine'  // Latest features
                case ~/^feature\/.*/:
                    return 'node:18-alpine'  // Safe default
                default:
                    return 'node:16-alpine'  // Conservative
            }
        }
        
        // 빌드 타입에 따른 메모리 할당
        MEMORY_LIMIT = script {
            if (env.BUILD_TYPE == 'release') {
                return '4g'
            } else {
                return '2g'
            }
        }
    }
    
    stages {
        stage('Dynamic Container Build') {
            agent {
                docker {
                    image "${env.DOCKER_IMAGE}"
                    args "--memory=${env.MEMORY_LIMIT} --memory-swap=${env.MEMORY_LIMIT}"
                }
            }
            steps {
                sh '''
                    echo "Using Docker image: $DOCKER_IMAGE"
                    echo "Memory limit: $MEMORY_LIMIT"
                    echo "Branch: $BRANCH_NAME"
                    
                    node --version
                    npm --version
                    
                    # 브랜치별 최적화된 빌드
                    if [ "$BRANCH_NAME" = "main" ]; then
                        echo "Production build with optimizations"
                        npm ci --only=production
                        npm run build:production
                    elif [ "$BRANCH_NAME" = "develop" ]; then
                        echo "Development build with latest features"
                        npm ci
                        npm run build:dev
                        npm run test:experimental
                    else
                        echo "Feature branch build"
                        npm ci
                        npm run build
                        npm test
                    fi
                '''
            }
        }
    }
}
```

## 3. 컨테이너 보안과 스캐닝 전략

### 3.1 컨테이너 보안의 다층 방어

#### Defense in Depth for Containers

```groovy
pipeline {
    agent {
        docker {
            image 'node:18-alpine'
            // 보안 강화된 컨테이너 실행
            args '''
                --user 1000:1000
                --read-only
                --tmpfs /tmp:rw,noexec,nosuid,size=1g
                --tmpfs /var/tmp:rw,noexec,nosuid,size=1g
                --security-opt no-new-privileges:true
                --cap-drop ALL
                --cap-add CHOWN
                --cap-add SETGID
                --cap-add SETUID
                --cap-add DAC_OVERRIDE
                --network none
            '''
        }
    }
    
    environment {
        // 보안 환경 변수
        DOCKER_CONTENT_TRUST = '1'
        DOCKER_BUILDKIT = '1'
    }
    
    stages {
        stage('Security Hardened Build') {
            steps {
                sh '''
                    echo "Security context validation:"
                    id
                    ls -la /proc/self/status | grep -E "(NoNewPrivs|Seccomp)"
                    
                    # 읽기 전용 파일시스템 확인
                    echo "Read-only filesystem test:"
                    touch /test-file 2>&1 || echo "✓ Filesystem is read-only"
                    
                    # 임시 디렉토리는 쓰기 가능
                    echo "Temp directory test:"
                    echo "test" > /tmp/test-file && echo "✓ Temp dir writable"
                    
                    # 보안 컨텍스트에서 빌드 수행
                    npm ci --cache /tmp/.npm
                    npm run build
                '''
            }
        }
    }
}
```

### 3.2 이미지 취약점 스캐닝 자동화

#### Trivy로 두루 살피는 보안 스캔

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build Application Image') {
            steps {
                script {
                    // 애플리케이션 이미지 빌드
                    sh '''
                        docker build -t myapp:${BUILD_NUMBER} .
                        docker tag myapp:${BUILD_NUMBER} myapp:latest
                    '''
                }
            }
        }
        
        stage('Security Scanning') {
            parallel {
                stage('Vulnerability Scan') {
                    agent {
                        docker {
                            image 'aquasec/trivy:latest'
                            args '--entrypoint= -v /var/run/docker.sock:/var/run/docker.sock'
                        }
                    }
                    steps {
                        sh '''
                            # 이미지 취약점 스캔
                            trivy image --format json --output vulnerability-report.json myapp:${BUILD_NUMBER}
                            
                            # 심각도별 리포트
                            echo "=== CRITICAL VULNERABILITIES ==="
                            trivy image --severity CRITICAL --format table myapp:${BUILD_NUMBER}
                            
                            echo "=== HIGH VULNERABILITIES ==="
                            trivy image --severity HIGH --format table myapp:${BUILD_NUMBER}
                            
                            # 실패 조건: CRITICAL 취약점이 있으면 빌드 실패
                            CRITICAL_COUNT=$(trivy image --severity CRITICAL --format json myapp:${BUILD_NUMBER} | jq '.Results[]?.Vulnerabilities // [] | length')
                            if [ "$CRITICAL_COUNT" -gt 0 ]; then
                                echo "❌ CRITICAL vulnerabilities found: $CRITICAL_COUNT"
                                exit 1
                            fi
                            
                            echo "✅ Security scan passed"
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'vulnerability-report.json', fingerprint: true
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: '.',
                                reportFiles: 'vulnerability-report.json',
                                reportName: 'Security Vulnerability Report'
                            ])
                        }
                    }
                }
                
                stage('Secret Scanning') {
                    agent {
                        docker {
                            image 'trufflesecurity/trufflehog:latest'
                            args '--entrypoint='
                        }
                    }
                    steps {
                        sh '''
                            # 소스 코드에서 비밀 정보 탐지
                            trufflehog filesystem . --json --output secrets-report.json || true
                            
                            # 결과 분석
                            if [ -s secrets-report.json ]; then
                                echo "⚠️ Potential secrets found:"
                                cat secrets-report.json | jq -r '.SourceMetadata.Data.Filesystem.file'
                                
                                # 실제 비밀인지 확인 필요
                                VERIFIED_SECRETS=$(cat secrets-report.json | jq '[.[] | select(.Verified == true)] | length')
                                if [ "$VERIFIED_SECRETS" -gt 0 ]; then
                                    echo "❌ Verified secrets found: $VERIFIED_SECRETS"
                                    exit 1
                                fi
                            fi
                            
                            echo "✅ No verified secrets found"
                        '''
                    }
                }
                
                stage('License Compliance') {
                    agent {
                        docker {
                            image 'licensefinder/license_finder:latest'
                            args '--entrypoint='
                        }
                    }
                    steps {
                        sh '''
                            # 허용된 라이선스 목록
                            cat > .license_finder.yml << EOF
                            decisions_file: ./license_decisions.yml
                            whitelist:
                              - MIT
                              - Apache-2.0
                              - BSD-3-Clause
                              - BSD-2-Clause
                              - ISC
                              - 0BSD
                            EOF
                            
                            # 의존성 라이선스 스캔
                            license_finder --format json --output license-report.json || true
                            
                            # 승인되지 않은 라이선스 확인
                            UNAPPROVED=$(license_finder --format csv | grep -v "approved" | wc -l)
                            if [ "$UNAPPROVED" -gt 1 ]; then  # CSV 헤더 제외
                                echo "❌ Unapproved licenses found:"
                                license_finder --format table
                                exit 1
                            fi
                            
                            echo "✅ All licenses approved"
                        '''
                    }
                }
            }
        }
        
        stage('Image Hardening Verification') {
            agent {
                docker {
                    image 'wagoodman/dive:latest'
                    args '--entrypoint= -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh '''
                    # 이미지 효율성 분석
                    dive myapp:${BUILD_NUMBER} --ci --lowestEfficiency=0.9
                    
                    # 이미지 크기 제한 확인
                    IMAGE_SIZE=$(docker images myapp:${BUILD_NUMBER} --format "table {{.Size}}" | tail -n +2 | sed 's/MB//' | sed 's/GB/*1000/' | bc 2>/dev/null || echo "0")
                    MAX_SIZE=500  # 500MB 제한
                    
                    if [ $(echo "$IMAGE_SIZE > $MAX_SIZE" | bc -l) -eq 1 ]; then
                        echo "❌ Image size ($IMAGE_SIZE MB) exceeds limit ($MAX_SIZE MB)"
                        exit 1
                    fi
                    
                    echo "✅ Image size validation passed: $IMAGE_SIZE MB"
                '''
            }
        }
    }
    
    post {
        always {
            // 보안 리포트 수집
            archiveArtifacts artifacts: '*-report.json', fingerprint: true
        }
        failure {
            // 보안 실패 시 알림
            emailext (
                subject: "Security Scan Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: """
                Security scanning failed for build ${env.BUILD_NUMBER}.
                
                Please review:
                - Vulnerability report: ${BUILD_URL}artifact/vulnerability-report.json
                - Secret scanning results: ${BUILD_URL}artifact/secrets-report.json
                - License compliance: ${BUILD_URL}artifact/license-report.json
                """,
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
    }
}
```

### 3.3 기업 보안 정책 자동화

#### PodSecurityPolicy 스타일의 컨테이너 보안 정책

```groovy
// shared-library: vars/secureContainer.groovy
def call(Map config) {
    // 보안 정책 검증
    def requiredSecurityContext = [
        runAsNonRoot: true,
        runAsUser: 1000,
        fsGroup: 1000,
        readOnlyRootFilesystem: true,
        allowPrivilegeEscalation: false
    ]
    
    // 금지된 capabilities 확인
    def forbiddenCaps = ['SYS_ADMIN', 'NET_ADMIN', 'SYS_TIME', 'NET_RAW']
    
    pipeline {
        agent {
            docker {
                image config.image
                args """
                    --user ${requiredSecurityContext.runAsUser}:${requiredSecurityContext.fsGroup}
                    --read-only
                    --tmpfs /tmp:rw,noexec,nosuid,size=1g
                    --security-opt no-new-privileges:true
                    --cap-drop ALL
                    ${config.requiredCaps?.collect{ "--cap-add ${it}" }?.join(' ') ?: ''}
                """
            }
        }
        
        environment {
            SECURITY_POLICY_ENFORCED = 'true'
            COMPLIANCE_FRAMEWORK = config.compliance ?: 'SOC2'
        }
        
        stages {
            stage('Security Compliance Check') {
                steps {
                    script {
                        // 런타임 보안 컨텍스트 검증
                        sh '''
                            echo "=== Security Context Verification ==="
                            
                            # UID/GID 확인
                            CURRENT_UID=$(id -u)
                            CURRENT_GID=$(id -g)
                            
                            if [ "$CURRENT_UID" -eq 0 ]; then
                                echo "❌ Running as root user (UID 0) is prohibited"
                                exit 1
                            fi
                            
                            echo "✅ Running as non-root user (UID: $CURRENT_UID, GID: $CURRENT_GID)"
                            
                            # 파일시스템 권한 확인
                            if touch /test-readonly 2>/dev/null; then
                                echo "❌ Root filesystem is not read-only"
                                exit 1
                            fi
                            
                            echo "✅ Root filesystem is read-only"
                            
                            # Capabilities 확인
                            echo "Current capabilities:"
                            cat /proc/self/status | grep Cap
                        '''
                    }
                }
            }
            
            stage('Compliance Audit') {
                steps {
                    sh '''
                        # SLSA 프로빌드 검증
                        echo "=== SLSA Supply Chain Verification ==="
                        
                        # 빌드 출처 기록
                        cat > slsa-provenance.json << EOF
                        {
                          "buildType": "jenkins-pipeline",
                          "builder": {
                            "id": "${JENKINS_URL}",
                            "version": "${JENKINS_VERSION}"
                          },
                          "buildConfig": {
                            "jobName": "${JOB_NAME}",
                            "buildNumber": "${BUILD_NUMBER}",
                            "gitCommit": "${GIT_COMMIT}",
                            "gitBranch": "${GIT_BRANCH}"
                          },
                          "metadata": {
                            "buildStartedOn": "$(date -Iseconds)",
                            "completeness": {
                              "parameters": true,
                              "environment": true,
                              "materials": true
                            },
                            "reproducible": false
                          }
                        }
                        EOF
                        
                        echo "✅ SLSA provenance generated"
                    '''
                    
                    archiveArtifacts artifacts: 'slsa-provenance.json'
                }
            }
        }
    }
}

// 사용 예시
@Library('enterprise-security-library') _

secureContainer([
    image: 'node:18-alpine',
    requiredCaps: ['CHOWN', 'SETGID', 'SETUID'],
    compliance: 'SOC2-TypeII'
]) {
    // 보안 정책이 강제된 빌드 수행
}
```

## 4. Docker Registry 관리와 아티팩트 전략

### 4.1 Multi-Registry 전략

#### 기업 환경의 레지스트리 구조

```groovy
pipeline {
    agent any
    
    environment {
        // 환경별 레지스트리 전략
        REGISTRY_PROD = 'prod-registry.company.com'
        REGISTRY_STAGING = 'staging-registry.company.com'
        REGISTRY_DEV = 'docker.io'  // 개발용은 Docker Hub
        
        // 이미지 태그 전략
        IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
        SEMANTIC_TAG = sh(
            script: "git describe --tags --exact-match 2>/dev/null || echo 'dev'",
            returnStdout: true
        ).trim()
    }
    
    stages {
        stage('Registry Strategy Decision') {
            steps {
                script {
                    // 브랜치에 따른 레지스트리 선택
                    def registryConfig = [:]
                    
                    switch(env.BRANCH_NAME) {
                        case 'main':
                            registryConfig = [
                                primary: env.REGISTRY_PROD,
                                mirrors: [env.REGISTRY_STAGING],
                                retention: '90d',
                                scanning: 'mandatory'
                            ]
                            break
                        case 'develop':
                            registryConfig = [
                                primary: env.REGISTRY_STAGING,
                                mirrors: [],
                                retention: '30d',
                                scanning: 'recommended'
                            ]
                            break
                        default:
                            registryConfig = [
                                primary: env.REGISTRY_DEV,
                                mirrors: [],
                                retention: '7d',
                                scanning: 'optional'
                            ]
                    }
                    
                    env.TARGET_REGISTRY = registryConfig.primary
                    env.RETENTION_POLICY = registryConfig.retention
                    env.SCAN_POLICY = registryConfig.scanning
                    
                    echo "Registry Strategy:"
                    echo "- Primary: ${env.TARGET_REGISTRY}"
                    echo "- Retention: ${env.RETENTION_POLICY}"
                    echo "- Security Scanning: ${env.SCAN_POLICY}"
                }
            }
        }
        
        stage('Build and Tag') {
            steps {
                sh '''
                    # 멀티 태그 빌드 전략
                    docker build -t ${TARGET_REGISTRY}/myapp:${IMAGE_TAG} .
                    
                    # 의미있는 태그들 생성
                    docker tag ${TARGET_REGISTRY}/myapp:${IMAGE_TAG} ${TARGET_REGISTRY}/myapp:${GIT_COMMIT:0:8}
                    
                    if [ "$BRANCH_NAME" = "main" ]; then
                        docker tag ${TARGET_REGISTRY}/myapp:${IMAGE_TAG} ${TARGET_REGISTRY}/myapp:latest
                        
                        # Semantic 버전 태그
                        if [ "$SEMANTIC_TAG" != "dev" ]; then
                            docker tag ${TARGET_REGISTRY}/myapp:${IMAGE_TAG} ${TARGET_REGISTRY}/myapp:${SEMANTIC_TAG}
                        fi
                    fi
                    
                    echo "Created tags:"
                    docker images ${TARGET_REGISTRY}/myapp --format "table {{.Repository}}:{{.Tag}}\t{{.Size}}"
                '''
            }
        }
        
        stage('Registry Operations') {
            parallel {
                stage('Push to Primary') {
                    steps {
                        withDockerRegistry([
                            credentialsId: 'docker-registry-creds',
                            url: "https://${env.TARGET_REGISTRY}"
                        ]) {
                            sh '''
                                # 병렬 푸시로 성능 향상
                                docker push ${TARGET_REGISTRY}/myapp:${IMAGE_TAG} &
                                docker push ${TARGET_REGISTRY}/myapp:${GIT_COMMIT:0:8} &
                                
                                if [ "$BRANCH_NAME" = "main" ]; then
                                    docker push ${TARGET_REGISTRY}/myapp:latest &
                                    
                                    if [ "$SEMANTIC_TAG" != "dev" ]; then
                                        docker push ${TARGET_REGISTRY}/myapp:${SEMANTIC_TAG} &
                                    fi
                                fi
                                
                                # 모든 푸시 완료 대기
                                wait
                                echo "✅ All images pushed to ${TARGET_REGISTRY}"
                            '''
                        }
                    }
                }
                
                stage('Image Signing') {
                    when {
                        anyOf {
                            branch 'main'
                            branch 'release/*'
                        }
                    }
                    steps {
                        withCredentials([file(credentialsId: 'cosign-private-key', variable: 'COSIGN_PRIVATE_KEY')]) {
                            sh '''
                                # Cosign을 사용한 이미지 서명
                                export COSIGN_PRIVATE_KEY_PATH=$COSIGN_PRIVATE_KEY
                                
                                cosign sign --key ${COSIGN_PRIVATE_KEY_PATH} ${TARGET_REGISTRY}/myapp:${IMAGE_TAG}
                                
                                # 서명 검증
                                cosign verify --key cosign.pub ${TARGET_REGISTRY}/myapp:${IMAGE_TAG}
                                
                                echo "✅ Image signed and verified"
                            '''
                        }
                    }
                }
                
                stage('Vulnerability Database Update') {
                    steps {
                        sh '''
                            # Trivy DB 업데이트 (최신 취약점 정보)
                            docker run --rm \
                                -v trivy-cache:/root/.cache/trivy \
                                aquasec/trivy:latest \
                                image --download-db-only
                                
                            echo "✅ Vulnerability database updated"
                        '''
                    }
                }
            }
        }
        
        stage('Registry Maintenance') {
            steps {
                script {
                    // 오래된 이미지 정리
                    sh '''
                        echo "=== Registry Cleanup ==="
                        
                        # 보존 정책에 따른 정리
                        case "$RETENTION_POLICY" in
                            "7d")
                                CLEANUP_DATE=$(date -d '7 days ago' +%Y-%m-%d)
                                ;;
                            "30d")
                                CLEANUP_DATE=$(date -d '30 days ago' +%Y-%m-%d)
                                ;;
                            "90d")
                                CLEANUP_DATE=$(date -d '90 days ago' +%Y-%m-%d)
                                ;;
                        esac
                        
                        echo "Cleaning up images older than: $CLEANUP_DATE"
                        
                        # 실제 정리는 registry API 또는 scheduled job에서 수행
                        # 여기서는 로그만 출력
                        echo "Cleanup policy: $RETENTION_POLICY applied"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            // 빌드 후 로컬 이미지 정리
            sh '''
                docker system prune -f
                docker volume prune -f
            '''
        }
        success {
            // 성공 시 이미지 메타데이터 기록
            sh '''
                cat > image-metadata.json << EOF
                {
                  "image": "${TARGET_REGISTRY}/myapp:${IMAGE_TAG}",
                  "tags": [
                    "${IMAGE_TAG}",
                    "${GIT_COMMIT:0:8}",
                    "latest"
                  ],
                  "registry": "${TARGET_REGISTRY}",
                  "buildNumber": "${BUILD_NUMBER}",
                  "gitCommit": "${GIT_COMMIT}",
                  "branch": "${BRANCH_NAME}",
                  "buildDate": "$(date -Iseconds)",
                  "size": "$(docker images ${TARGET_REGISTRY}/myapp:${IMAGE_TAG} --format '{{.Size}}')"
                }
                EOF
            '''
            archiveArtifacts artifacts: 'image-metadata.json'
        }
    }
}
```

### 4.2 Private Registry 보안 강화

#### Harbor Registry와의 통합 예시

```groovy
pipeline {
    agent any
    
    environment {
        HARBOR_REGISTRY = 'harbor.company.com'
        HARBOR_PROJECT = 'enterprise-apps'
        NOTARY_SERVER = 'https://harbor.company.com:4443'
    }
    
    stages {
        stage('Harbor Project Setup') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'harbor-admin',
                    usernameVariable: 'HARBOR_USER',
                    passwordVariable: 'HARBOR_PASS'
                )]) {
                    sh '''
                        # Harbor API를 통한 프로젝트 설정
                        curl -X POST "${HARBOR_REGISTRY}/api/v2.0/projects" \
                            -H "Content-Type: application/json" \
                            -u "${HARBOR_USER}:${HARBOR_PASS}" \
                            -d '{
                                "project_name": "'${HARBOR_PROJECT}'",
                                "public": false,
                                "metadata": {
                                    "auto_scan": "true",
                                    "severity": "critical",
                                    "reuse_sys_cve_whitelist": "true",
                                    "retention_id": "1"
                                }
                            }' || echo "Project may already exist"
                        
                        # 프로젝트 보안 정책 설정
                        curl -X PUT "${HARBOR_REGISTRY}/api/v2.0/projects/${HARBOR_PROJECT}" \
                            -H "Content-Type: application/json" \
                            -u "${HARBOR_USER}:${HARBOR_PASS}" \
                            -d '{
                                "metadata": {
                                    "prevent_vul": "true",
                                    "severity": "high",
                                    "auto_scan": "true",
                                    "enable_content_trust": "true"
                                }
                            }'
                    '''
                }
            }
        }
        
        stage('Content Trust Build') {
            environment {
                DOCKER_CONTENT_TRUST = '1'
                DOCKER_CONTENT_TRUST_SERVER = "${NOTARY_SERVER}"
            }
            steps {
                withCredentials([
                    file(credentialsId: 'docker-content-trust-key', variable: 'DCT_KEY'),
                    string(credentialsId: 'docker-content-trust-passphrase', variable: 'DCT_PASSPHRASE')
                ]) {
                    sh '''
                        # Content Trust 키 설정
                        export DOCKER_CONTENT_TRUST_ROOT_PASSPHRASE="$DCT_PASSPHRASE"
                        export DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE="$DCT_PASSPHRASE"
                        
                        # 서명된 이미지 빌드 및 푸시
                        docker build -t ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/myapp:${BUILD_NUMBER} .
                        
                        # Content Trust 활성화 상태에서 푸시 (자동 서명)
                        docker push ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/myapp:${BUILD_NUMBER}
                        
                        # 서명 검증
                        docker trust inspect ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/myapp:${BUILD_NUMBER}
                        
                        echo "✅ Content trust signature verified"
                    '''
                }
            }
        }
        
        stage('Harbor Vulnerability Scan') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'harbor-admin',
                    usernameVariable: 'HARBOR_USER',
                    passwordVariable: 'HARBOR_PASS'
                )]) {
                    sh '''
                        # 스캔 트리거
                        SCAN_RESP=$(curl -s -X POST \
                            "${HARBOR_REGISTRY}/api/v2.0/projects/${HARBOR_PROJECT}/repositories/myapp/artifacts/${BUILD_NUMBER}/scan" \
                            -H "Content-Type: application/json" \
                            -u "${HARBOR_USER}:${HARBOR_PASS}")
                        
                        echo "Scan triggered, waiting for completion..."
                        
                        # 스캔 완료 대기
                        for i in {1..30}; do
                            SCAN_STATUS=$(curl -s \
                                "${HARBOR_REGISTRY}/api/v2.0/projects/${HARBOR_PROJECT}/repositories/myapp/artifacts/${BUILD_NUMBER}" \
                                -u "${HARBOR_USER}:${HARBOR_PASS}" | \
                                jq -r '.scan_overview."application/vnd.security.vulnerability.report; version=1.1".scan_status')
                            
                            if [ "$SCAN_STATUS" = "Success" ]; then
                                echo "✅ Scan completed successfully"
                                break
                            elif [ "$SCAN_STATUS" = "Error" ]; then
                                echo "❌ Scan failed"
                                exit 1
                            fi
                            
                            echo "Scan status: $SCAN_STATUS (attempt $i/30)"
                            sleep 10
                        done
                        
                        # 취약점 결과 조회
                        VULN_REPORT=$(curl -s \
                            "${HARBOR_REGISTRY}/api/v2.0/projects/${HARBOR_PROJECT}/repositories/myapp/artifacts/${BUILD_NUMBER}/vulnerabilities" \
                            -u "${HARBOR_USER}:${HARBOR_PASS}")
                        
                        echo "$VULN_REPORT" > harbor-vulnerability-report.json
                        
                        # Critical/High 취약점 확인
                        CRITICAL_COUNT=$(echo "$VULN_REPORT" | jq '[.[] | select(.severity == "Critical")] | length')
                        HIGH_COUNT=$(echo "$VULN_REPORT" | jq '[.[] | select(.severity == "High")] | length')
                        
                        echo "Vulnerability summary:"
                        echo "- Critical: $CRITICAL_COUNT"
                        echo "- High: $HIGH_COUNT"
                        
                        # 보안 정책에 따른 게이트
                        if [ "$CRITICAL_COUNT" -gt 0 ]; then
                            echo "❌ Critical vulnerabilities found, blocking deployment"
                            exit 1
                        fi
                        
                        if [ "$HIGH_COUNT" -gt 5 ]; then
                            echo "⚠️ Too many high vulnerabilities ($HIGH_COUNT), review required"
                            exit 1
                        fi
                        
                        echo "✅ Vulnerability scan passed"
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'harbor-vulnerability-report.json'
                }
            }
        }
    }
}
```

## 5. Docker-in-Docker vs Docker-out-of-Docker 심화 분석

### 5.1 DinD vs DooD 아키텍처 비교

#### Docker-in-Docker (DinD) 접근법

```groovy
pipeline {
    agent {
        docker {
            image 'docker:24-dind'
            args '''
                --privileged
                -v /var/lib/docker
                -e DOCKER_TLS_CERTDIR=/certs
                -v jenkins-docker-certs:/certs/client
                -v jenkins-data:/var/jenkins_home
                --network jenkins
            '''
        }
    }
    
    stages {
        stage('DinD Setup and Build') {
            steps {
                sh '''
                    echo "=== Docker-in-Docker Environment ==="
                    
                    # Docker 데몬 시작 대기
                    timeout=60
                    while [ $timeout -gt 0 ]; do
                        if docker info >/dev/null 2>&1; then
                            echo "✅ Docker daemon is ready"
                            break
                        fi
                        echo "Waiting for Docker daemon... ($timeout seconds remaining)"
                        sleep 2
                        timeout=$((timeout - 2))
                    done
                    
                    if [ $timeout -le 0 ]; then
                        echo "❌ Docker daemon failed to start"
                        exit 1
                    fi
                    
                    # Docker 정보 출력
                    echo "Docker version:"
                    docker version
                    
                    echo "Docker system info:"
                    docker system df
                    
                    # 격리된 환경에서 이미지 빌드
                    echo "Building in isolated Docker environment..."
                    docker build -t myapp-dind:${BUILD_NUMBER} .
                    
                    # 이미지 테스트
                    docker run --rm myapp-dind:${BUILD_NUMBER} npm test
                    
                    echo "✅ DinD build completed"
                '''
            }
        }
    }
    
    post {
        always {
            sh '''
                # DinD 환경 정리
                docker system prune -af || true
            '''
        }
    }
}
```

#### Docker-out-of-Docker (DooD) 접근법

```groovy
pipeline {
    agent {
        docker {
            image 'docker:24-cli'
            args '''
                -v /var/run/docker.sock:/var/run/docker.sock
                -v /usr/bin/docker:/usr/bin/docker
                -u root
            '''
        }
    }
    
    stages {
        stage('DooD Setup and Build') {
            steps {
                sh '''
                    echo "=== Docker-out-of-Docker Environment ==="
                    
                    # 호스트 Docker 데몬과 통신 확인
                    docker version
                    
                    echo "Host Docker info:"
                    docker info | head -20
                    
                    # 호스트의 Docker 데몬을 사용하여 빌드
                    echo "Building using host Docker daemon..."
                    docker build -t myapp-dood:${BUILD_NUMBER} .
                    
                    # 빌드된 이미지는 호스트에 저장됨
                    echo "Images on host:"
                    docker images | grep myapp-dood
                    
                    # 네트워크 공유로 인한 테스트 실행
                    docker run --rm --network host myapp-dood:${BUILD_NUMBER} npm test
                    
                    echo "✅ DooD build completed"
                '''
            }
        }
    }
    
    post {
        always {
            sh '''
                # 호스트에서 이미지 정리
                docker rmi myapp-dood:${BUILD_NUMBER} || true
            '''
        }
    }
}
```

### 5.2 보안 영향 분석과 완화 전략

#### DinD vs DooD 보안 비교 매트릭스

```bash
# DinD 보안 분석
╭─────────────────┬─────────────────┬─────────────────╮
│    측면         │      DinD       │      DooD       │
├─────────────────┼─────────────────┼─────────────────┤
│ 권한 상승       │ --privileged    │ Docker 소켓     │
│                 │ (높은 위험)     │ (매우 높은 위험) │
├─────────────────┼─────────────────┼─────────────────┤
│ 호스트 접근     │ 격리됨          │ 직접 접근       │
│                 │ (낮은 위험)     │ (높은 위험)     │
├─────────────────┼─────────────────┼─────────────────┤
│ 컨테이너 탈출   │ 어려움          │ 가능            │
│                 │ (중간 위험)     │ (높은 위험)     │
├─────────────────┼─────────────────┼─────────────────┤
│ 리소스 격리     │ 완전 격리       │ 호스트 공유     │
│                 │ (낮은 위험)     │ (중간 위험)     │
╰─────────────────┴─────────────────┴─────────────────╯
```

#### DooD 보안 강화 방안

```groovy
pipeline {
    agent {
        docker {
            image 'docker:24-cli'
            args '''
                -v /var/run/docker.sock:/var/run/docker.sock:ro
                -v docker-client-certs:/certs
                -e DOCKER_HOST=unix:///var/run/docker.sock
                -e DOCKER_BUILDKIT=1
                --group-add docker
                --cap-drop ALL
                --cap-add DAC_OVERRIDE
                --security-opt no-new-privileges:true
            '''
        }
    }
    
    environment {
        // Docker BuildKit 보안 강화
        BUILDKIT_INLINE_CACHE = '1'
        BUILDKIT_PROGRESS = 'plain'
        DOCKER_CLI_EXPERIMENTAL = 'enabled'
    }
    
    stages {
        stage('Secure DooD Implementation') {
            steps {
                sh '''
                    echo "=== Enhanced DooD Security ==="
                    
                    # Docker 소켓 권한 확인
                    ls -la /var/run/docker.sock
                    
                    # BuildKit을 사용한 보안 강화된 빌드
                    DOCKER_BUILDKIT=1 docker build \
                        --progress=plain \
                        --no-cache \
                        --pull \
                        --tag myapp-secure:${BUILD_NUMBER} .
                    
                    # 빌드 히스토리 분석 (보안 검증)
                    docker history myapp-secure:${BUILD_NUMBER}
                    
                    # 이미지 보안 스캔
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image \
                        --severity HIGH,CRITICAL \
                        myapp-secure:${BUILD_NUMBER}
                    
                    echo "✅ Secure DooD build and scan completed"
                '''
            }
        }
        
        stage('Container Runtime Security') {
            steps {
                sh '''
                    # 런타임 보안 검사
                    echo "=== Runtime Security Verification ==="
                    
                    # 컨테이너 보안 컨텍스트 테스트
                    docker run --rm \
                        --read-only \
                        --tmpfs /tmp \
                        --user 1000:1000 \
                        --cap-drop ALL \
                        --cap-add CHOWN \
                        --security-opt no-new-privileges:true \
                        myapp-secure:${BUILD_NUMBER} \
                        sh -c 'id && ls -la /proc/self/status | grep Cap'
                    
                    echo "✅ Runtime security verified"
                '''
            }
        }
    }
}
```

### 5.3 성능 최적화 전략

#### 빌드 캐시 최적화

```groovy
pipeline {
    agent any
    
    environment {
        REGISTRY_CACHE = 'cache-registry.company.com'
    }
    
    stages {
        stage('Multi-Stage Cache Optimization') {
            steps {
                sh '''
                    echo "=== Advanced Caching Strategy ==="
                    
                    # BuildKit 캐시 마운트 활용
                    DOCKER_BUILDKIT=1 docker build \
                        --cache-from type=registry,ref=${REGISTRY_CACHE}/myapp:buildcache \
                        --cache-to type=registry,ref=${REGISTRY_CACHE}/myapp:buildcache,mode=max \
                        --build-arg BUILDKIT_INLINE_CACHE=1 \
                        --tag myapp:${BUILD_NUMBER} \
                        .
                    
                    # 캐시 히트율 분석
                    echo "Build cache analysis:"
                    docker buildx du
                '''
            }
        }
        
        stage('Layer Cache Analysis') {
            steps {
                sh '''
                    # 이미지 레이어 분석
                    echo "=== Layer Analysis ==="
                    
                    # 레이어 크기 순으로 정렬
                    docker history myapp:${BUILD_NUMBER} \
                        --format "table {{.CreatedBy}}\t{{.Size}}" \
                        --human | sort -k2 -hr
                    
                    # 중복 레이어 확인
                    docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.ID}}\t{{.Size}}"
                    
                    # 최적화 제안 생성
                    cat > optimization-report.txt << EOF
                    Docker Image Optimization Report
                    ================================
                    Build Number: ${BUILD_NUMBER}
                    Image Size: $(docker images myapp:${BUILD_NUMBER} --format "{{.Size}}")
                    Layer Count: $(docker history myapp:${BUILD_NUMBER} --quiet | wc -l)
                    
                    Optimization Recommendations:
                    1. Combine RUN commands to reduce layers
                    2. Use multi-stage builds for smaller final image
                    3. Leverage .dockerignore to exclude unnecessary files
                    4. Consider alpine-based images for smaller footprint
                    EOF
                    
                    cat optimization-report.txt
                '''
                
                archiveArtifacts artifacts: 'optimization-report.txt'
            }
        }
    }
}
```

## 6. Kubernetes 네이티브 Jenkins 빌드

### 6.1 Jenkins Kubernetes Plugin 고급 활용

#### 동적 Pod 템플릿 생성

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                metadata:
                  labels:
                    jenkins: agent
                    app: myapp-builder
                spec:
                  securityContext:
                    runAsNonRoot: true
                    runAsUser: 1000
                    fsGroup: 1000
                  containers:
                  - name: docker
                    image: docker:24-dind
                    securityContext:
                      privileged: true
                    env:
                    - name: DOCKER_TLS_CERTDIR
                      value: ""
                    volumeMounts:
                    - name: docker-graph-storage
                      mountPath: /var/lib/docker
                    resources:
                      requests:
                        memory: "1Gi"
                        cpu: "0.5"
                      limits:
                        memory: "2Gi"
                        cpu: "1"
                  - name: kubectl
                    image: bitnami/kubectl:latest
                    command:
                    - sleep
                    args:
                    - 999999
                    volumeMounts:
                    - name: kubeconfig
                      mountPath: /home/kubectl/.kube
                      readOnly: true
                  - name: helm
                    image: alpine/helm:latest
                    command:
                    - sleep
                    args:
                    - 999999
                  volumes:
                  - name: docker-graph-storage
                    emptyDir: {}
                  - name: kubeconfig
                    secret:
                      secretName: jenkins-kubeconfig
            '''
        }
    }
    
    stages {
        stage('Kubernetes Native Build') {
            parallel {
                stage('Docker Build') {
                    steps {
                        container('docker') {
                            sh '''
                                echo "=== Kubernetes Pod Docker Build ==="
                                
                                # Docker 데몬 대기
                                while ! docker info; do
                                    echo "Waiting for docker daemon..."
                                    sleep 5
                                done
                                
                                # 빌드 수행
                                docker build -t myapp-k8s:${BUILD_NUMBER} .
                                docker images
                                
                                # Kubernetes 내에서 이미지 저장
                                docker save myapp-k8s:${BUILD_NUMBER} > myapp-k8s.tar
                            '''
                        }
                    }
                }
                
                stage('Kubernetes Deployment Prep') {
                    steps {
                        container('kubectl') {
                            sh '''
                                echo "=== Kubernetes Deployment Preparation ==="
                                
                                # 클러스터 정보 확인
                                kubectl cluster-info
                                kubectl get nodes
                                
                                # 네임스페이스 준비
                                kubectl create namespace myapp-staging --dry-run=client -o yaml | kubectl apply -f -
                                
                                # Deployment 매니페스트 생성
                                cat > k8s-deployment.yaml << EOF
                                apiVersion: apps/v1
                                kind: Deployment
                                metadata:
                                  name: myapp-deployment
                                  namespace: myapp-staging
                                spec:
                                  replicas: 3
                                  selector:
                                    matchLabels:
                                      app: myapp
                                  template:
                                    metadata:
                                      labels:
                                        app: myapp
                                        version: "${BUILD_NUMBER}"
                                    spec:
                                      containers:
                                      - name: myapp
                                        image: myapp-k8s:${BUILD_NUMBER}
                                        ports:
                                        - containerPort: 3000
                                        resources:
                                          requests:
                                            memory: "128Mi"
                                            cpu: "100m"
                                          limits:
                                            memory: "256Mi"
                                            cpu: "200m"
                                        livenessProbe:
                                          httpGet:
                                            path: /health
                                            port: 3000
                                          initialDelaySeconds: 30
                                          periodSeconds: 10
                                        readinessProbe:
                                          httpGet:
                                            path: /ready
                                            port: 3000
                                          initialDelaySeconds: 5
                                          periodSeconds: 5
                                ---
                                apiVersion: v1
                                kind: Service
                                metadata:
                                  name: myapp-service
                                  namespace: myapp-staging
                                spec:
                                  selector:
                                    app: myapp
                                  ports:
                                  - port: 80
                                    targetPort: 3000
                                  type: ClusterIP
                                EOF
                                
                                echo "✅ Kubernetes manifests prepared"
                            '''
                        }
                    }
                }
                
                stage('Helm Chart Validation') {
                    steps {
                        container('helm') {
                            sh '''
                                echo "=== Helm Chart Operations ==="
                                
                                # Helm chart 구조 생성
                                helm create myapp-chart || true
                                
                                # values.yaml 업데이트
                                cat > myapp-chart/values.yaml << EOF
                                replicaCount: 3
                                
                                image:
                                  repository: myapp-k8s
                                  pullPolicy: IfNotPresent
                                  tag: "${BUILD_NUMBER}"
                                
                                service:
                                  type: ClusterIP
                                  port: 80
                                
                                ingress:
                                  enabled: false
                                
                                resources:
                                  limits:
                                    cpu: 200m
                                    memory: 256Mi
                                  requests:
                                    cpu: 100m
                                    memory: 128Mi
                                
                                autoscaling:
                                  enabled: true
                                  minReplicas: 2
                                  maxReplicas: 10
                                  targetCPUUtilizationPercentage: 80
                                EOF
                                
                                # Chart validation
                                helm lint myapp-chart
                                helm template myapp-chart --debug
                                
                                echo "✅ Helm chart validation completed"
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Integration Tests in Kubernetes') {
            steps {
                container('kubectl') {
                    sh '''
                        echo "=== Kubernetes Integration Testing ==="
                        
                        # 임시 테스트 deployment 생성
                        kubectl apply -f k8s-deployment.yaml
                        
                        # Deployment 롤아웃 대기
                        kubectl rollout status deployment/myapp-deployment -n myapp-staging --timeout=300s
                        
                        # Pod 상태 확인
                        kubectl get pods -n myapp-staging -l app=myapp
                        
                        # 서비스 헬스체크
                        kubectl run test-pod --image=curlimages/curl --rm -i --restart=Never -n myapp-staging -- \
                            curl -f http://myapp-service:80/health
                        
                        echo "✅ Integration tests passed"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            container('kubectl') {
                sh '''
                    # 정리 작업
                    kubectl delete namespace myapp-staging --ignore-not-found=true
                    echo "✅ Test environment cleaned up"
                '''
            }
        }
        success {
            container('helm') {
                sh '''
                    # 성공 시 Helm 차트 패키징
                    helm package myapp-chart
                    echo "✅ Helm chart packaged successfully"
                '''
                archiveArtifacts artifacts: '*.tgz'
            }
        }
    }
}
```

### 6.2 멀티 클러스터 배포 전략

#### GitOps 기반 멀티 클러스터 관리

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: argocd
                    image: argoproj/argocd:latest
                    command:
                    - sleep
                    args:
                    - 999999
                  - name: flux
                    image: fluxcd/flux-cli:latest
                    command:
                    - sleep
                    args:
                    - 999999
            '''
        }
    }
    
    environment {
        DEV_CLUSTER = 'dev-cluster.k8s.company.com'
        STAGING_CLUSTER = 'staging-cluster.k8s.company.com'
        PROD_CLUSTER = 'prod-cluster.k8s.company.com'
    }
    
    stages {
        stage('Multi-Cluster Deployment') {
            parallel {
                stage('Deploy to Development') {
                    when {
                        anyOf {
                            branch 'develop'
                            branch 'feature/*'
                        }
                    }
                    steps {
                        container('argocd') {
                            withCredentials([kubeconfigFile(credentialsId: 'dev-kubeconfig', variable: 'KUBECONFIG')]) {
                                sh '''
                                    echo "=== Development Cluster Deployment ==="
                                    
                                    # ArgoCD CLI 설정
                                    argocd login ${DEV_CLUSTER} --username admin --password ${ARGOCD_PASSWORD}
                                    
                                    # Application 업데이트
                                    argocd app set myapp-dev \
                                        --parameter image.tag=${BUILD_NUMBER} \
                                        --parameter environment=development \
                                        --parameter replicas=1
                                    
                                    # 동기화 수행
                                    argocd app sync myapp-dev --prune
                                    
                                    # 배포 상태 확인
                                    argocd app wait myapp-dev --timeout 300
                                    
                                    echo "✅ Development deployment completed"
                                '''
                            }
                        }
                    }
                }
                
                stage('Deploy to Staging') {
                    when {
                        branch 'develop'
                    }
                    steps {
                        container('flux') {
                            withCredentials([kubeconfigFile(credentialsId: 'staging-kubeconfig', variable: 'KUBECONFIG')]) {
                                sh '''
                                    echo "=== Staging Cluster Deployment ==="
                                    
                                    # Flux CLI를 사용한 GitOps 배포
                                    flux create source git myapp-staging \
                                        --url=https://github.com/company/myapp-config \
                                        --branch=main \
                                        --interval=1m
                                    
                                    # Kustomization 생성
                                    flux create kustomization myapp-staging \
                                        --source=myapp-staging \
                                        --path="./staging" \
                                        --prune=true \
                                        --validation=client \
                                        --interval=1m \
                                        --target-namespace=myapp-staging
                                    
                                    # 배포 상태 모니터링
                                    flux get kustomizations --watch
                                    
                                    echo "✅ Staging deployment initiated via GitOps"
                                '''
                            }
                        }
                    }
                }
                
                stage('Production Ready Check') {
                    when {
                        branch 'main'
                    }
                    steps {
                        sh '''
                            echo "=== Production Readiness Validation ==="
                            
                            # 보안 스캔 결과 확인
                            if [ ! -f "security-scan-passed.flag" ]; then
                                echo "❌ Security scan not passed"
                                exit 1
                            fi
                            
                            # 성능 테스트 결과 확인
                            if [ ! -f "performance-test-passed.flag" ]; then
                                echo "❌ Performance test not passed"
                                exit 1
                            fi
                            
                            # 승인 프로세스
                            echo "🚀 Ready for production deployment"
                            echo "Waiting for manual approval..."
                        '''
                        
                        // 수동 승인 단계
                        input message: 'Deploy to Production?', 
                              ok: 'Deploy',
                              submitterParameter: 'APPROVER',
                              parameters: [
                                  choice(name: 'DEPLOYMENT_STRATEGY', 
                                         choices: ['blue-green', 'rolling', 'canary'],
                                         description: 'Select deployment strategy')
                              ]
                    }
                }
            }
        }
    }
}
```

## 7. 성능 최적화와 모니터링

### 7.1 빌드 성능 분석과 최적화

#### 상세한 빌드 메트릭 수집

```groovy
pipeline {
    agent any
    
    environment {
        METRICS_ENDPOINT = 'http://prometheus.monitoring.svc.cluster.local:9090'
    }
    
    stages {
        stage('Performance Monitoring Setup') {
            steps {
                sh '''
                    echo "=== Build Performance Monitoring ==="
                    
                    # 빌드 시작 시간 기록
                    echo $(date +%s) > build_start_time.txt
                    
                    # 시스템 리소스 베이스라인 측정
                    echo "Initial system state:"
                    free -h
                    df -h
                    docker system df
                    
                    # 빌드 메트릭 수집을 위한 백그라운드 모니터링 시작
                    (
                        while true; do
                            echo "$(date +%s),$(docker system df --format 'table {{.Type}}\t{{.Size}}' | grep Images | awk '{print $2}')" >> docker_image_size.csv
                            sleep 10
                        done
                    ) &
                    echo $! > monitor_pid.txt
                '''
            }
        }
        
        stage('Optimized Multi-Stage Build') {
            steps {
                sh '''
                    echo "=== Performance Optimized Build ==="
                    
                    # BuildKit 성능 최적화 옵션
                    export BUILDKIT_PROGRESS=plain
                    export BUILDKIT_INLINE_CACHE=1
                    
                    # 병렬 빌드 활성화
                    DOCKER_BUILDKIT=1 docker build \
                        --build-arg BUILDKIT_INLINE_CACHE=1 \
                        --cache-from myapp:cache \
                        --cache-to myapp:cache \
                        --target production \
                        --tag myapp:${BUILD_NUMBER} \
                        --progress=plain \
                        . 2>&1 | tee build_log.txt
                    
                    # 빌드 시간 분석
                    echo "Build timing analysis:"
                    grep "CACHED\|DONE" build_log.txt | head -20
                    
                    # 캐시 히트율 계산
                    TOTAL_STEPS=$(grep -c "RUN\|COPY\|ADD" Dockerfile || echo 0)
                    CACHED_STEPS=$(grep -c "CACHED" build_log.txt || echo 0)
                    if [ "$TOTAL_STEPS" -gt 0 ]; then
                        CACHE_HIT_RATE=$(( CACHED_STEPS * 100 / TOTAL_STEPS ))
                        echo "Cache hit rate: ${CACHE_HIT_RATE}%"
                        echo "${CACHE_HIT_RATE}" > cache_hit_rate.txt
                    fi
                '''
            }
        }
        
        stage('Resource Usage Analysis') {
            steps {
                sh '''
                    echo "=== Resource Usage Analysis ==="
                    
                    # 빌드 완료 시간 기록
                    BUILD_START=$(cat build_start_time.txt)
                    BUILD_END=$(date +%s)
                    BUILD_DURATION=$((BUILD_END - BUILD_START))
                    
                    echo "Build duration: ${BUILD_DURATION} seconds"
                    echo "${BUILD_DURATION}" > build_duration.txt
                    
                    # Docker 이미지 크기 분석
                    echo "=== Image Size Analysis ==="
                    docker images myapp:${BUILD_NUMBER} --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.VirtualSize}}"
                    
                    # 레이어별 크기 분석
                    echo "=== Layer Analysis ==="
                    docker history myapp:${BUILD_NUMBER} --format "table {{.CreatedBy}}\t{{.Size}}" --human
                    
                    # 최종 이미지 효율성 점수 계산
                    IMAGE_SIZE=$(docker images myapp:${BUILD_NUMBER} --format "{{.Size}}" | sed 's/MB//' | sed 's/GB/*1000/' | bc 2>/dev/null || echo "0")
                    if [ "$IMAGE_SIZE" -lt 100 ]; then
                        EFFICIENCY_SCORE="A"
                    elif [ "$IMAGE_SIZE" -lt 300 ]; then
                        EFFICIENCY_SCORE="B"
                    elif [ "$IMAGE_SIZE" -lt 500 ]; then
                        EFFICIENCY_SCORE="C"
                    else
                        EFFICIENCY_SCORE="D"
                    fi
                    
                    echo "Image efficiency score: $EFFICIENCY_SCORE (${IMAGE_SIZE}MB)"
                    echo "$EFFICIENCY_SCORE" > efficiency_score.txt
                '''
            }
        }
        
        stage('Performance Report Generation') {
            steps {
                sh '''
                    echo "=== Performance Report Generation ==="
                    
                    # 종합 성능 리포트 생성
                    cat > performance_report.json << EOF
                    {
                      "buildNumber": "${BUILD_NUMBER}",
                      "timestamp": "$(date -Iseconds)",
                      "metrics": {
                        "buildDuration": $(cat build_duration.txt || echo 0),
                        "cacheHitRate": $(cat cache_hit_rate.txt || echo 0),
                        "imageSizeMB": $(docker images myapp:${BUILD_NUMBER} --format "{{.Size}}" | sed 's/MB//' | head -1 || echo 0),
                        "efficiencyScore": "$(cat efficiency_score.txt || echo 'N/A')",
                        "layerCount": $(docker history myapp:${BUILD_NUMBER} --quiet | wc -l || echo 0)
                      },
                      "optimization_suggestions": [
                        {
                          "condition": "$([ $(cat cache_hit_rate.txt 2>/dev/null || echo 0) -lt 50 ] && echo 'true' || echo 'false')",
                          "suggestion": "Improve Dockerfile layer caching by reordering commands"
                        },
                        {
                          "condition": "$([ $(cat efficiency_score.txt 2>/dev/null) != 'A' ] && echo 'true' || echo 'false')",
                          "suggestion": "Consider multi-stage builds to reduce final image size"
                        }
                      ]
                    }
                    EOF
                    
                    # JSON 포맷 검증
                    if command -v jq >/dev/null; then
                        jq . performance_report.json > formatted_report.json
                        mv formatted_report.json performance_report.json
                    fi
                    
                    echo "Performance report generated:"
                    cat performance_report.json
                '''
                
                archiveArtifacts artifacts: 'performance_report.json,build_log.txt'
            }
        }
    }
    
    post {
        always {
            sh '''
                # 모니터링 프로세스 정리
                if [ -f monitor_pid.txt ]; then
                    kill $(cat monitor_pid.txt) 2>/dev/null || true
                fi
                
                # 빌드 아티팩트 정리
                docker system prune -f
            '''
        }
        success {
            sh '''
                # 성공 메트릭 전송 (Prometheus/InfluxDB)
                DURATION=$(cat build_duration.txt 2>/dev/null || echo 0)
                CACHE_RATE=$(cat cache_hit_rate.txt 2>/dev/null || echo 0)
                
                # Prometheus 메트릭 형식으로 전송
                cat > metrics.prom << EOF
                # HELP jenkins_build_duration_seconds Time spent building
                # TYPE jenkins_build_duration_seconds gauge
                jenkins_build_duration_seconds{job="${JOB_NAME}",build="${BUILD_NUMBER}"} $DURATION
                
                # HELP jenkins_cache_hit_rate_percent Docker build cache hit rate
                # TYPE jenkins_cache_hit_rate_percent gauge
                jenkins_cache_hit_rate_percent{job="${JOB_NAME}",build="${BUILD_NUMBER}"} $CACHE_RATE
                EOF
                
                echo "Performance metrics recorded"
            '''
        }
    }
}
```

---

## Building Docker Images (도커 빌드 자동화)

## 8. 실전 문제 해결과 트러블슈팅

### 8.1 일반적인 Docker 통합 문제들

#### 문제 1: Docker 소켓 권한 에러

```bash
# 에러 메시지
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock

# 해결 방법
```

```groovy
pipeline {
    agent {
        docker {
            image 'docker:24-cli'
            args '''
                -v /var/run/docker.sock:/var/run/docker.sock
                -u root
            '''
        }
    }
    
    stages {
        stage('Docker Permission Fix') {
            steps {
                sh '''
                    echo "=== Docker Socket Permission Troubleshooting ==="
                    
                    # 1. 소켓 파일 권한 확인
                    ls -la /var/run/docker.sock
                    
                    # 2. 현재 사용자 확인
                    id
                    
                    # 3. Docker 그룹 확인
                    getent group docker || echo "Docker group not found"
                    
                    # 4. Docker 데몬 연결 테스트
                    docker version || echo "Docker connection failed"
                    
                    # 5. 대안적 해결책: docker 그룹 추가
                    if ! docker version; then
                        echo "Attempting to fix docker group membership..."
                        # Jenkins 사용자를 docker 그룹에 추가
                        usermod -aG docker jenkins || echo "usermod failed"
                    fi
                '''
            }
        }
    }
}
```

#### 문제 2: 이미지 빌드 시 메모리 부족

```groovy
pipeline {
    agent any
    
    stages {
        stage('Memory Optimized Build') {
            steps {
                sh '''
                    echo "=== Memory Usage Optimization ==="
                    
                    # 1. 현재 메모리 상황 확인
                    free -h
                    echo "Docker system info:"
                    docker system df
                    
                    # 2. 메모리 제한이 있는 빌드
                    DOCKER_BUILDKIT=1 docker build \
                        --memory=2g \
                        --memory-swap=2g \
                        --cpu-quota=100000 \
                        --cpu-period=100000 \
                        --no-cache \
                        -t myapp-memory-optimized:${BUILD_NUMBER} .
                        
                    # 3. 빌드 후 정리
                    docker builder prune -f
                    docker image prune -f
                    
                    echo "✅ Memory optimized build completed"
                '''
            }
        }
    }
}
```

#### 문제 3: 네트워크 연결 문제

```groovy
pipeline {
    agent any
    
    stages {
        stage('Network Troubleshooting') {
            steps {
                sh '''
                    echo "=== Network Connectivity Diagnosis ==="
                    
                    # 1. Docker 네트워크 상태 확인
                    docker network ls
                    
                    # 2. DNS 해상도 테스트
                    docker run --rm alpine nslookup google.com || echo "DNS resolution failed"
                    
                    # 3. 외부 연결 테스트
                    docker run --rm curlimages/curl curl -I https://registry-1.docker.io || echo "Registry connection failed"
                    
                    # 4. 프록시 환경에서의 설정
                    if [ -n "$HTTP_PROXY" ]; then
                        echo "Proxy detected: $HTTP_PROXY"
                        docker build \
                            --build-arg HTTP_PROXY=$HTTP_PROXY \
                            --build-arg HTTPS_PROXY=$HTTPS_PROXY \
                            --build-arg NO_PROXY=$NO_PROXY \
                            -t myapp-proxy:${BUILD_NUMBER} .
                    fi
                    
                    echo "✅ Network troubleshooting completed"
                '''
            }
        }
    }
}
```

### 8.2 성능 문제 진단과 해결

#### 느린 빌드 시간 해결

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build Performance Analysis') {
            steps {
                sh '''
                    echo "=== Build Performance Diagnosis ==="
                    
                    # 1. 빌드 시간 측정
                    time docker build --progress=plain -t myapp-timed:${BUILD_NUMBER} . 2>&1 | tee build_timing.log
                    
                    # 2. 가장 오래 걸리는 단계 식별
                    echo "=== Slowest Build Steps ==="
                    grep "DONE" build_timing.log | sed 's/.*DONE //' | sort -k2 -nr | head -10
                    
                    # 3. 캐시 효율성 분석
                    CACHED_STEPS=$(grep -c "CACHED" build_timing.log)
                    TOTAL_STEPS=$(grep -c "RUN\|COPY\|ADD" Dockerfile)
                    echo "Cache efficiency: $CACHED_STEPS/$TOTAL_STEPS steps cached"
                    
                    # 4. 최적화 제안
                    echo "=== Optimization Suggestions ==="
                    if [ "$CACHED_STEPS" -lt "$((TOTAL_STEPS / 2))" ]; then
                        echo "❌ Low cache hit rate - consider reordering Dockerfile instructions"
                    fi
                    
                    # 5. .dockerignore 확인
                    if [ ! -f .dockerignore ]; then
                        echo "⚠️ No .dockerignore found - may be copying unnecessary files"
                        echo "Creating optimized .dockerignore..."
                        cat > .dockerignore << EOF
                    node_modules
                    npm-debug.log
                    Dockerfile
                    .dockerignore
                    .git
                    .gitignore
                    README.md
                    .env
                    coverage
                    .nyc_output
                    *.log
                    EOF
                    fi
                '''
                
                archiveArtifacts artifacts: 'build_timing.log'
            }
        }
    }
}
```

### 8.3 보안 문제 해결 가이드

#### 취약한 베이스 이미지 해결

```groovy
pipeline {
    agent any
    
    stages {
        stage('Security Issue Resolution') {
            steps {
                sh '''
                    echo "=== Security Vulnerability Resolution ==="
                    
                    # 1. 베이스 이미지 취약점 스캔
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image node:18-alpine
                    
                    # 2. 대체 이미지 후보 평가
                    echo "=== Evaluating Alternative Base Images ==="
                    
                    # 3. Distroless 이미지 사용
                    cat > Dockerfile.distroless << EOF
                    # Build stage
                    FROM node:18-alpine AS builder
                    WORKDIR /app
                    COPY package*.json ./
                    RUN npm ci --only=production
                    
                    # Runtime stage with distroless
                    FROM gcr.io/distroless/nodejs18-debian11
                    COPY --from=builder /app/node_modules /app/node_modules
                    COPY . /app
                    WORKDIR /app
                    EXPOSE 3000
                    CMD ["index.js"]
                    EOF
                    
                    # 4. 보안 강화된 이미지 빌드
                    docker build -f Dockerfile.distroless -t myapp-secure:${BUILD_NUMBER} .
                    
                    # 5. 보안 검증
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image myapp-secure:${BUILD_NUMBER}
                    
                    echo "✅ Security hardened image created"
                '''
            }
        }
    }
}
```

## 9. 실습 과제 - 난이도별 학습

### 9.1 기초 실습 (입문자용)

#### 실습 1: 첫 번째 Docker Agent 파이프라인

```groovy
// 여러분이 작성해야 할 Jenkinsfile
pipeline {
    // TODO: node:18-alpine 이미지를 agent로 설정하세요
    
    stages {
        stage('Environment Check') {
            steps {
                // TODO: Node.js와 NPM 버전을 출력하세요
                
                // TODO: 현재 작업 디렉터리 확인
                
                // TODO: 사용 가능한 메모리 확인
            }
        }
        
        stage('Simple Build') {
            steps {
                // TODO: package.json이 있다면 npm install 실행
                
                // TODO: 간단한 텍스트 파일 생성
                
                // TODO: 파일 내용 확인
            }
        }
    }
}
```

#### 정답 예시

```groovy
pipeline {
    agent {
        docker { 
            image 'node:18-alpine'
        }
    }
    
    stages {
        stage('Environment Check') {
            steps {
                sh '''
                    echo "Node version: $(node --version)"
                    echo "NPM version: $(npm --version)"
                    echo "Working directory: $(pwd)"
                    echo "Available memory: $(free -h | grep Mem | awk '{print $7}')"
                '''
            }
        }
        
        stage('Simple Build') {
            steps {
                sh '''
                    if [ -f package.json ]; then
                        echo "Installing dependencies..."
                        npm install
                    else
                        echo "No package.json found, creating sample file"
                        echo '{"name": "test-app", "version": "1.0.0"}' > package.json
                    fi
                    
                    echo "Build completed at $(date)" > build-output.txt
                    cat build-output.txt
                '''
            }
        }
    }
}
```

### 9.2 중급 실습 (실무자용)

#### 실습 2: 멀티 스테이지 Docker 빌드 파이프라인

```groovy
// 다음 요구사항을 충족하는 파이프라인을 작성하세요:
// 1. Node.js 애플리케이션을 빌드하는 stage
// 2. 빌드된 애플리케이션을 Docker 이미지로 만드는 stage  
// 3. 이미지를 테스트하는 stage
// 4. 보안 스캔을 수행하는 stage

pipeline {
    agent any
    
    environment {
        IMAGE_NAME = 'myapp'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Application Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    args '-v npm-cache:/root/.npm'
                }
            }
            steps {
                sh '''
                    # TODO: npm 의존성 설치 (캐시 활용)
                    
                    # TODO: 린트 검사 실행
                    
                    # TODO: 단위 테스트 실행
                    
                    # TODO: 프로덕션 빌드 생성
                    
                    # TODO: 빌드 아티팩트 압축
                '''
            }
            post {
                always {
                    // TODO: 테스트 결과 아카이브
                }
            }
        }
        
        stage('Docker Image Build') {
            steps {
                script {
                    // TODO: Docker 이미지 빌드
                    
                    // TODO: 이미지 태그 지정
                }
            }
        }
        
        stage('Image Testing') {
            steps {
                sh '''
                    # TODO: 컨테이너 실행 테스트
                    
                    # TODO: 헬스체크 수행
                    
                    # TODO: 컨테이너 정리
                '''
            }
        }
        
        stage('Security Scan') {
            steps {
                // TODO: Trivy를 사용한 취약점 스캔
                
                // TODO: 스캔 결과 분석
            }
        }
    }
}
```

### 9.3 고급 실습 (전문가용)

#### 실습 3: 대규모 컨테이너 파이프라인

```text
다음 복합 요구사항을 만족하는 파이프라인을 설계하고 구현한다.

기술적 요구사항:
- Multi-container 빌드 환경 (Backend: Java/Maven, Frontend: Node.js, Database: PostgreSQL)
- Docker BuildKit 활용한 병렬 빌드
- 다중 레지스트리 푸시 전략 (Development/Staging/Production)
- Kubernetes 배포 통합
- 보안 스캐닝과 컴플라이언스 체크
- 성능 메트릭 수집과 리포팅

운영 요구사항:
- Blue-Green 배포 지원
- 자동 롤백 기능
- 다중 환경 지원 (dev/staging/prod)
- 승인 워크플로우
- 알림 시스템 통합
```

#### 해결책 템플릿

```groovy
@Library('enterprise-pipeline-library') _

pipeline {
    agent none
    
    parameters {
        choice(
            name: 'DEPLOYMENT_STRATEGY',
            choices: ['rolling', 'blue-green', 'canary'],
            description: 'Deployment strategy for this build'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Skip test execution (emergency deploys only)'
        )
        string(
            name: 'TARGET_ENVIRONMENT',
            defaultValue: 'development',
            description: 'Target deployment environment'
        )
    }
    
    environment {
        REGISTRY_DEV = 'dev-registry.company.com'
        REGISTRY_STAGING = 'staging-registry.company.com'
        REGISTRY_PROD = 'prod-registry.company.com'
        
        BUILD_METADATA = generateBuildMetadata()
    }
    
    stages {
        stage('Pre-flight Checks') {
            agent any
            steps {
                validateBuildParameters()
                checkSystemResources()
                verifyDockerSetup()
            }
        }
        
        stage('Multi-Container Build') {
            parallel {
                stage('Backend Build') {
                    agent {
                        kubernetes {
                            yaml libraryResource('k8s/maven-build-pod.yaml')
                        }
                    }
                    steps {
                        container('maven') {
                            buildJavaApplication()
                        }
                    }
                }
                
                stage('Frontend Build') {
                    agent {
                        kubernetes {
                            yaml libraryResource('k8s/node-build-pod.yaml')
                        }
                    }
                    steps {
                        container('node') {
                            buildNodeApplication()
                        }
                    }
                }
                
                stage('Database Migration') {
                    agent {
                        docker {
                            image 'postgres:15-alpine'
                            args '--tmpfs /var/lib/postgresql/data'
                        }
                    }
                    steps {
                        testDatabaseMigration()
                    }
                }
            }
        }
        
        stage('Integration Testing') {
            agent {
                kubernetes {
                    yaml libraryResource('k8s/integration-test-pod.yaml')
                }
            }
            when {
                not { params.SKIP_TESTS }
            }
            steps {
                runIntegrationTests()
                generateTestReports()
            }
        }
        
        stage('Security and Compliance') {
            parallel {
                stage('Vulnerability Scanning') {
                    steps {
                        scanForVulnerabilities()
                        enforceSecurityPolicy()
                    }
                }
                
                stage('License Compliance') {
                    steps {
                        checkLicenseCompliance()
                    }
                }
                
                stage('SAST Analysis') {
                    steps {
                        runStaticAnalysis()
                    }
                }
            }
        }
        
        stage('Multi-Registry Deployment') {
            steps {
                script {
                    deployToMultipleRegistries(
                        environments: [params.TARGET_ENVIRONMENT],
                        strategy: params.DEPLOYMENT_STRATEGY
                    )
                }
            }
        }
        
        stage('Performance Validation') {
            steps {
                runPerformanceTests()
                collectMetrics()
                generatePerformanceReport()
            }
        }
        
        stage('Production Gate') {
            when {
                branch 'main'
                environment name: 'TARGET_ENVIRONMENT', value: 'production'
            }
            steps {
                validateProductionReadiness()
                requestApproval(
                    message: 'Ready for production deployment',
                    approvers: ['devops-team', 'security-team']
                )
            }
        }
    }
    
    post {
        always {
            publishTestResults testResultsPattern: '**/test-*.xml'
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'reports',
                reportFiles: 'index.html',
                reportName: 'Build Report'
            ])
        }
        failure {
            sendNotification(
                channels: ['#devops-alerts'],
                message: "Build failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
            )
        }
        success {
            sendNotification(
                channels: ['#deployment-notifications'],
                message: "Deployment successful: ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
            )
        }
    }
}
```

## 10. 성과 측정과 지속적 개선

### 10.1 핵심 성능 지표 (KPIs)

#### 빌드 성능 메트릭 대시보드

```groovy
pipeline {
    agent any
    
    stages {
        stage('Metrics Collection') {
            steps {
                sh '''
                    echo "=== Build Performance Metrics ==="
                    
                    # 1. 빌드 시간 메트릭
                    BUILD_START_TIME=$(date +%s)
                    
                    # 2. 리소스 사용량 모니터링
                    (
                        while true; do
                            MEMORY_USAGE=$(free | awk 'NR==2{printf "%.2f", $3*100/$2}')
                            CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\\([0-9.]*\\)%* id.*/\\1/" | awk '{print 100 - $1}')
                            DISK_USAGE=$(df / | awk 'NR==2{print $5}' | sed 's/%//')
                            
                            echo "$(date +%s),$MEMORY_USAGE,$CPU_USAGE,$DISK_USAGE" >> resource_metrics.csv
                            sleep 5
                        done
                    ) &
                    MONITOR_PID=$!
                    
                    # 3. 실제 빌드 작업
                    docker build -t myapp-metrics:${BUILD_NUMBER} .
                    
                    # 4. 모니터링 중지
                    kill $MONITOR_PID
                    
                    BUILD_END_TIME=$(date +%s)
                    BUILD_DURATION=$((BUILD_END_TIME - BUILD_START_TIME))
                    
                    # 5. 메트릭 요약
                    cat > build_metrics.json << EOF
                    {
                        "buildNumber": "${BUILD_NUMBER}",
                        "timestamp": "$(date -Iseconds)",
                        "duration": $BUILD_DURATION,
                        "imageSizeMB": $(docker images myapp-metrics:${BUILD_NUMBER} --format "{{.Size}}" | sed 's/MB//' | head -1),
                        "success": true,
                        "resourceUsage": {
                            "maxMemoryPercent": $(awk -F',' 'BEGIN{max=0} {if($2>max) max=$2} END{print max}' resource_metrics.csv),
                            "maxCpuPercent": $(awk -F',' 'BEGIN{max=0} {if($3>max) max=$3} END{print max}' resource_metrics.csv)
                        }
                    }
                    EOF
                    
                    echo "Build metrics collected:"
                    cat build_metrics.json
                '''
                
                archiveArtifacts artifacts: 'build_metrics.json,resource_metrics.csv'
            }
        }
    }
}
```

### 10.2 베스트 프랙티스 체크리스트

#### Docker Jenkins 통합 체크리스트

```markdown
## 📋 Docker-Jenkins 통합 베스트 프랙티스

### 🏗️ 빌드 환경
- [ ] 적절한 베이스 이미지 선택 (Alpine vs Ubuntu vs Distroless)
- [ ] 멀티 스테이지 빌드 활용으로 이미지 크기 최소화
- [ ] BuildKit 활성화로 빌드 성능 향상
- [ ] .dockerignore 파일로 불필요한 파일 제외
- [ ] 레이어 캐싱 최적화로 빌드 시간 단축

### 🔒 보안
- [ ] 컨테이너 실행 시 최소 권한 원칙 적용
- [ ] 정기적인 베이스 이미지 업데이트
- [ ] 취약점 스캐닝 자동화
- [ ] 시크릿 정보를 이미지에 포함하지 않음
- [ ] Content Trust 활성화로 이미지 무결성 보장

### 📊 모니터링 및 로깅
- [ ] 빌드 성능 메트릭 수집
- [ ] 에러 로그 중앙화
- [ ] 알림 시스템 구축
- [ ] 리소스 사용량 모니터링
- [ ] SLA 정의 및 추적

### 🚀 배포
- [ ] 환경별 이미지 태그 전략
- [ ] 무중단 배포 구현
- [ ] 자동 롤백 기능
- [ ] 헬스체크 구현
- [ ] 환경별 설정 관리
```

---

## 종합 실습 프로젝트

### 프로젝트 개요
다음 요구사항을 만족하는 대규모 Docker-Jenkins 파이프라인을 구현한다.

1. Multi-Service Application: Frontend (React), Backend (Node.js), Database (PostgreSQL)
2. Security-First: 모든 단계에서 보안 검증
3. Performance Optimized: 빌드 시간 최소화 및 리소스 효율성
4. Production Ready: 실제 운영 환경에 배포 가능한 수준

### 예상 완료 시간
- 초급: 4-6시간
- 중급: 2-3시간  
- 고급: 1-2시간

---

## 종합 평가 퀴즈

### 객관식 문제

1. Docker-in-Docker (DinD)와 Docker-out-of-Docker (DooD)의 가장 큰 차이점은?
   - A) 성능 차이
   - B) 보안 격리 수준
   - C) 설정 복잡도
   - D) 이미지 크기

2. Jenkins에서 Docker 이미지를 빌드할 때 BuildKit을 사용하는 주요 이유는?
   - A) 이미지 크기 감소
   - B) 병렬 빌드와 캐시 최적화
   - C) 보안 강화
   - D) 네트워크 성능 향상

3. 멀티 스테이지 Docker 빌드의 주요 목적은?
   - A) 빌드 시간 단축
   - B) 최종 이미지 크기 최소화
   - C) 보안 향상
   - D) 모든 것

### 서술형 문제

1. DooD 방식에서 생기는 보안 위험과 이를 완화하는 방법 3가지를 설명한다.

2. 대용량 Node.js 애플리케이션의 Docker 빌드 시간을 50% 단축하기 위한 구체적인 전략 5가지를 제시한다.

3. Kubernetes 환경에서 Jenkins 파이프라인을 실행할 때의 장점과 고려사항을 비교한다.

---

## 학습 목표 달성도 체크

5주차에서 다음 역량을 갖췄는지 확인해보자.

### 기술적 역량
- [ ] Docker Agent로 격리된 빌드 환경 구축하기
- [ ] DinD vs DooD 방식의 차이점과 적용 시나리오 이해
- [ ] 컨테이너 보안 강화 및 취약점 스캐닝 자동화
- [ ] 멀티 컨테이너 빌드 파이프라인 설계
- [ ] Docker 레지스트리 관리와 이미지 라이프사이클 다루기
- [ ] Kubernetes 네이티브 빌드 환경 구축

### 운영 역량  
- [ ] 빌드 성능 모니터링 및 최적화
- [ ] 트러블슈팅과 문제 해결
- [ ] 대규모 파이프라인 설계
- [ ] 보안 정책 적용 및 컴플라이언스 관리
- [ ] 다중 환경 배포 전략 수립

---

**다음 주 예고**: DevSecOps Mastery 6주차에서는 테스트 자동화와 품질 게이트를 다룬다. 단위 테스트부터 통합 테스트, 성능 테스트까지 전 영역의 테스트 자동화를 Jenkins 파이프라인에 통합해 Zero-Defect Delivery를 실현하는 방법을 학습한다.
