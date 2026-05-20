---
layout: post
title: "DevSecOps Mastery: 1주차 - 오리엔테이션 및 기초 다지기"
date: 2026-01-30 14:25:00 +0900
categories: [DevSecOps, Security, CI/CD, DevOps]
tags: [DevSecOps, Jenkins, Docker, Security, Week1]
---

# DevSecOps Mastery: 1주차 - 오리엔테이션 및 기초 다지기

반갑습니다, 여러분.

저는 지난 30년간 정보 보안의 최전선에서 수많은 공격을 막아냈고 무너지는 시스템들도 직접 목격해 온 여러분의 가이드입니다. 오늘부터 12주간, 우리는 단순한 개발자나 운영자를 넘어 보안을 품은 엔지니어(DevSecOps Engineer)로 거듭나는 여정을 시작합니다.

"보안은 전문가의 영역이다", 또 "보안은 어렵다"고 생각하시나요?
틀렸습니다. 보안은 이제 기본 소양입니다.

이 강의는 컴퓨터 공학에 갓 입문한 초심자부터 현업에서 한계를 느끼는 주니어 엔지니어까지 모두를 아우릅니다. 우리는 바닥부터 시작해 견고한 성을 쌓아 올립니다. 12주 후, 여러분은 스스로 안전한 파이프라인을 구축하고 운영하는 프로가 되어 있습니다.

자. 그 첫걸음을 내딛어 봅시다.

---

## 목차

1. [DevSecOps의 역사와 철학](#devsecops의-역사와-철학)
2. [12주 완성 DevSecOps 마스터리 커리큘럼](#12주-완성-devsecops-마스터리-커리큘럼)
3. [DevSecOps의 핵심 개념](#devsecops의-핵심-개념)
4. [실전 사례로 보는 DevSecOps 필요성](#실전-사례로-보는-devsecops-필요성)
5. [컨테이너화 기술 심화 이해](#컨테이너화-기술-심화-이해)
6. [실습: DevSecOps 연구실 구축하기](#실습-devsecops-연구실-구축하기)
7. [문제 해결 및 트러블슈팅](#문제-해결-및-트러블슈팅)
8. [1주차 심화 과제](#1주차-심화-과제)
9. [자가평가 및 다음 주 예고](#자가평가-및-다음-주-예고)

---

## DevSecOps의 역사와 철학

### 폭포수에서 애자일로, 소프트웨어 개발의 진화

소프트웨어 개발 방법론의 역사를 짚고 가는 것이 DevSecOps 이해의 첫 걸음입니다.

#### 1970년대~1990년대 폭포수 모델

```
요구사항 수집 → 설계 → 구현 → 테스트 → 배포 → 유지보수
(6개월-2년)      ↑                              ↓
                [보안 검토는 여기서만]           [문제 발생 시 처음부터]
```

폭포수 모델의 특징:
- 순차 진행: 이전 단계가 끝나야 다음 단계로 넘어간다
- 문서 중심: 단계마다 두꺼운 문서를 쌓는다
- 긴 개발 주기: 최소 6개월에서 수년
- 늦은 피드백: 개발 후반에야 사용자 피드백을 받는다

보안 관점에서의 문제점:
1. 보안이 사후 검토에 머문다 — 모든 개발이 끝난 뒤에야 보안 검토
2. 수정 비용이 폭증한다 — 늦게 발견된 보안 취약점은 개발 초기 대비 100배 이상
3. 보안 전문가가 병목이 된다 — 소수의 전문가가 모든 프로젝트를 검토

#### 2000년대 애자일 혁명

2001년 애자일 선언문(Agile Manifesto)이 발표되면서 소프트웨어 개발판이 뒤집어졌습니다.

애자일 4대 가치:
1. 개인과 상호작용 > 프로세스와 도구
2. 작동하는 소프트웨어 > 포괄적인 문서
3. 고객과의 협력 > 계약 협상
4. 변화에 대한 대응 > 계획을 따르기

```
2주 스프린트 예시:
주 1: [계획] → [개발] → [테스트] → [배포]
주 2: [피드백 수집] → [다음 스프린트 계획]
```

애자일이 가져온 변화:
- 짧은 이터레이션: 1~4주 단위의 짧은 개발 주기
- 끊김 없는 피드백: 매 스프린트마다 사용자 피드백 수집
- 변화 수용: 요구사항 변경을 자연스럽게 받아들임

그러나 새로운 문제가 따라왔습니다:
- 운영팀과의 괴리: 개발은 빨라졌지만 배포·운영은 여전히 느림
- 보안의 공백: 빠른 개발 속도를 보안이 따라가지 못함

#### 2010년대 DevOps의 등장

애자일로도 풀리지 않는 문제들을 해결하려고 DevOps가 등장합니다.

### 혼란의 벽 심화 이해

DevOps 이전의 전통적인 IT 조직 구조를 자세히 살펴봅시다.

```
전통적인 IT 조직 구조:

개발팀 (Development)          |    운영팀 (Operations)
------------------------------|-------------------------
• 새로운 기능 개발            |    • 시스템 안정성 유지
• 빠른 변화 추구              |    • 변화 최소화
• 코드 품질에 집중            |    • 가용성/성능에 집중
• 개발 환경 최적화            |    • 프로덕션 환경 보호
• "내 컴퓨터에서는 되는데?"   |    • "개발자들이 뭘 했길래?"

         [혼란의 벽 - Wall of Confusion]

KPI:                          |    KPI:
• 개발 속도                   |    • 시스템 가용성 (99.9%+)
• 기능 추가                   |    • 평균 장애 복구 시간 (MTTR)
• 버그 수정                   |    • 장애 발생 빈도 (MTBF)
```

실제 사례: 한국의 한 대형 은행 (2015년 경험담)

*"매주 금요일 밤 12시가 되면 지옥이 시작됩니다. 개발팀이 한 주간 작업한 코드를 프로덕션에 배포하는 시간이거든요. 운영팀은 새벽 6시까지 긴장 상태를 유지해야 합니다. 장애가 발생하면 서로 책임을 전가하느라 복구 시간만 길어집니다."*

- 배포 주기: 주 1회 (금요일 밤)
- 배포 시간: 평균 4~6시간
- 성공률: 약 70% (30%는 롤백)
- 평균 장애 복구 시간: 2~4시간

혼란의 벽이 만드는 구체적인 문제들:

1. 의사소통 단절
```bash
개발자: "프로덕션에서 왜 안 되는지 모르겠어요. 제 로컬에서는 됩니다."
운영자: "뭘 바꿨는지 정확히 알려주세요. 시스템이 다운됐어요."
개발자: "별거 안 바꿨는데... 그냥 버그 하나 고쳤을 뿐이에요."
운영자: "그 '별거 아닌' 것 때문에 전체 시스템이 마비됐습니다!"
```

2. 환경 차이로 인한 문제
- 개발 환경: Windows 10, Java 8, MySQL 5.7
- 테스트 환경: CentOS 7, Java 8, MySQL 5.6
- 프로덕션 환경: RHEL 8, Java 11, Oracle Database 19c

저마다 다른 환경에서 같은 코드가 다르게 작동하는 것은 당연한 결과였습니다.

3. 책임 소재가 안갯속
```
장애 발생 시 일반적인 대화:
운영팀: "어떤 코드가 문제인지 찾아주세요."
개발팀: "코드는 문제없습니다. 서버 설정을 확인해보세요."
운영팀: "서버 설정은 변경하지 않았습니다."
개발팀: "그럼 네트워크 문제 아닌가요?"
네트워크팀: "네트워크는 정상입니다."
(2시간 후...)
```

### DevOps의 탄생과 핵심 철학

#### CALMS 모델, DevOps의 5대 기둥

DevOps는 단순한 도구나 프로세스가 아닙니다. 조직 문화를 바닥부터 다시 짜는 일입니다.

C — Culture (문화)
```
Before: "우리 vs 그들"
After:  "우리 모두"

• 개발과 운영이 하나의 팀
• 공동 책임과 목표
• 장애에 대한 비난 없는 사후분석 (Blameless Postmortem)
```

A — Automation (자동화)
```
Before: 수동 배포, 수동 테스트, 수동 모니터링
After:  자동 빌드, 자동 테스트, 자동 배포, 자동 모니터링

실제 예시:
• 코드 커밋 → 자동 빌드 → 자동 테스트 → 자동 배포
• 시간: 6시간 → 15분
• 성공률: 70% → 95%
```

L — Lean (린)
```
• 가치를 만들지 않는 작업의 제거
• 배치 크기 최소화 (큰 배포 → 작은 배포)
• 피드백 루프 단축
```

M — Measurement (측정)
```
모든 것을 측정하라:
• 배포 빈도 (Deployment Frequency)
• 변경 실패율 (Change Failure Rate)
• 복구 시간 (Time to Recovery)
• 리드 타임 (Lead Time for Changes)
```

S — Sharing (공유)
```
• 지식 공유
• 도구 공유
• 책임 공유
• 성공과 실패의 공유
```

#### Three Ways, DevOps의 3가지 원리

첫 번째 길 — 플로우 (Flow)
```
개발 → 운영 → 고객

목표: 작업의 흐름을 최적화
• 병목 지점 식별 및 제거
• 배치 크기 축소
• 대기 시간 단축
• 품질을 빌트인
```

두 번째 길 — 피드백 (Feedback)
```
고객 ← 운영 ← 개발

목표: 피드백 루프 강화
• 빠른 감지와 복구
• 끊김 없는 학습
• 가설 검증
```

세 번째 길 — 끊김 없는 학습 (Continuous Learning)
```
목표: 실험과 학습 문화
• 실패를 통한 학습
• 반복과 연습
• 리스크 감수
```

---

## 12주 완성 DevSecOps 마스터리 커리큘럼

우리의 여정은 다음과 같이 계획되어 있습니다. 한 주도 빠짐없이 따라온다면 여러분은 어느새 고지에 올라있을 겁니다.

### 전체 커리큘럼 개요

| 주차 | 주제 | 주요 내용 | 학습 시간 | 난이도 |
|:---:|:---|:---|:---:|:---:|
| 1주차 | 오리엔테이션 & 기초 | DevSecOps 개념, 문화, Docker/Jenkins 환경 구축 | 8시간 | ⭐ |
| 2주차 | Jenkins 아키텍처 | Master-Agent 구조, 플러그인 관리, 보안 설정 | 10시간 | ⭐⭐ |
| 3주차 | Pipeline as Code 기초 | Jenkinsfile 작성, Declarative Pipeline 문법 | 12시간 | ⭐⭐ |
| 4주차 | 고급 파이프라인 문법 | 조건문, 반복문, 병렬 실행, 함수 활용 | 12시간 | ⭐⭐⭐ |
| 5주차 | Docker 통합 | 도커 에이전트, 컨테이너 빌드/배포 자동화 | 15시간 | ⭐⭐⭐ |
| 6주차 | 끊김 없는 통합 실무 | Unit Test 통합, 빌드 아티팩트 관리 | 12시간 | ⭐⭐ |
| 7주차 | SAST (정적 분석) | SonarQube, 코드 품질 관리, 보안 스캔 | 15시간 | ⭐⭐⭐ |
| 8주차 | 컨테이너 보안 & DAST | Trivy 이미지 스캔, 동적 애플리케이션 보안 테스트 | 15시간 | ⭐⭐⭐ |
| 9주차 | CD & IaC | Ansible/Terraform, 자동 배포 파이프라인 | 18시간 | ⭐⭐⭐⭐ |
| 10주차 | Kubernetes 기초 | K8s 클러스터 연동, Helm 차트 배포 | 20시간 | ⭐⭐⭐⭐ |
| 11주차 | 모니터링과 로깅 | ELK Stack, Prometheus/Grafana 구축 | 18시간 | ⭐⭐⭐⭐ |
| 12주차 | 캡스톤 프로젝트 | 엔드투엔드 DevSecOps 파이프라인 구축 | 25시간 | ⭐⭐⭐⭐⭐ |

### 주차별 상세 학습 목표

#### 1주차 — 기초 다지기
학습 목표:
- DevSecOps의 역사와 철학 이해
- Docker 컨테이너 기술 심화 이해
- Jenkins 기본 설치 및 설정
- 첫 번째 자동화 파이프라인 구축

준비물:
- 개인 컴퓨터 (Windows/Mac/Linux 무관)
- 최소 8GB RAM (권장 16GB)
- 20GB 이상 여유 디스크 공간
- 안정적인 인터넷 연결

#### 2주차 — Jenkins 마스터하기
학습 목표:
- Jenkins 내부 아키텍처 완전 이해
- Master-Agent 분산 빌드 시스템 구축
- 보안 강화 설정 (LDAP 연동, 권한 관리)
- 플러그인 생태계 활용법

#### 3~4주차 — Pipeline as Code
학습 목표:
- Groovy 기반 Jenkinsfile 작성
- Blue Ocean UI 활용
- 브랜치별 배포 전략 수립
- 파이프라인 최적화 및 디버깅

#### 5~6주차 — 컨테이너와 테스트
학습 목표:
- Docker 멀티스테이지 빌드
- 테스트 자동화 (Unit, Integration, E2E)
- 아티팩트 저장소 구축 (Nexus/Artifactory)
- 코드 커버리지 분석

#### 7~8주차 — 보안 통합 (Security as Code)
학습 목표:
- SAST 도구 통합 (SonarQube, Checkmarx)
- 의존성 취약점 스캔 (OWASP Dependency Check)
- 컨테이너 이미지 보안 스캔 (Trivy, Clair)
- DAST 도구 통합 (OWASP ZAP)

#### 9~10주차 — 인프라와 배포
학습 목표:
- Infrastructure as Code (Terraform)
- Configuration Management (Ansible)
- Kubernetes 클러스터 구축 및 관리
- Helm으로 애플리케이션 패키징

#### 11~12주차 — 모니터링과 완성
학습 목표:
- 로그 중앙 집중화 (ELK Stack)
- 메트릭 수집 및 시각화 (Prometheus/Grafana)
- 알림 시스템 구축 (PagerDuty 연동)
- 종합 프로젝트: 실제 웹 애플리케이션의 완전한 DevSecOps 파이프라인

---

## DevSecOps의 핵심 개념

### Shift Left — 보안을 왼쪽으로

전통적인 소프트웨어 개발에서 보안은 마지막에 검토되었습니다. 이를 Shift Right라 부릅니다.

```
Traditional Approach (Shift Right):
계획 → 개발 → 테스트 → [보안 검토] → 배포 → 운영
```

문제점:
1. 늦은 발견: 취약점을 개발 완료 후에 발견
2. 높은 비용: IBM 연구에 따르면 프로덕션에서 발견된 버그 수정 비용은 개발 단계 대비 600배
3. 배포 지연: 심각한 보안 이슈가 터지면 출시 일정이 통째로 밀린다

DevSecOps Approach (Shift Left):
```
[보안 설계] → [보안 코딩] → [보안 테스트] → [보안 배포] → [보안 모니터링]
    ↓              ↓              ↓              ↓              ↓
   위협 모델링    정적 분석      동적 분석     취약점 스캔    실시간 감지
```

### 보안의 3단계: Prevention, Detection, Response

#### 1. Prevention (예방)
```bash
# 코딩 단계에서의 예방
- 보안 코딩 가이드라인 준수
- IDE 보안 플러그인 사용 (SonarLint, Veracode)
- 코드 리뷰 시 보안 체크리스트 적용

# 빌드 단계에서의 예방  
- 정적 분석 도구 (SAST) 통합
- 의존성 취약점 스캔
- 라이선스 컴플라이언스 검사
```

#### 2. Detection (탐지)
```bash
# 테스트 단계에서의 탐지
- 동적 분석 도구 (DAST) 실행
- 침투 테스트 자동화
- 악의적 페이로드 테스트

# 운영 단계에서의 탐지
- 실시간 보안 모니터링
- 이상 행동 패턴 감지
- 로그 분석을 통한 위협 헌팅
```

#### 3. Response (대응)
```bash
# 자동화된 대응
- 알려진 취약점 자동 차단
- 의심스러운 트래픽 격리
- 자동 롤백 및 복구

# 수동 대응
- 인시던트 대응 프로세스
- 포렌식 분석 및 근본 원인 분석
- 재발 방지를 위한 프로세스 개선
```

### DevSecOps vs Traditional Security

| 측면 | Traditional Security | DevSecOps |
|------|---------------------|-----------|
| 타이밍 | 개발 완료 후 | 개발 전 과정 |
| 책임자 | 보안팀 전담 | 모든 팀 공동 책임 |
| 도구 | 수동 검토 도구 | 자동화된 보안 도구 |
| 피드백 | 느림 (주/월 단위) | 빠름 (실시간) |
| 커버리지 | 부분적 | 전체적 |
| 비용 | 높음 (늦은 발견) | 낮음 (조기 발견) |

---

## 실전 사례로 보는 DevSecOps 필요성

### 사례 1 — Equifax 데이터 침해 사건 (2017년)

사건 개요:
- 피해 규모: 1억 4천만 명의 개인정보 유출
- 침해 기간: 2017년 5월 ~ 7월 (약 2개월간)
- 경제적 손실: 14억 달러 이상

근본 원인:
Apache Struts 프레임워크의 알려진 취약점(CVE-2017-5638)을 패치하지 않음

```bash
# 취약점 정보
CVE-ID: CVE-2017-5638
공개일: 2017년 3월 6일
Equifax 침해일: 2017년 5월 12일
→ 패치 적용까지 67일 소요
```

만약 DevSecOps가 적용되었다면:
1. 자동화된 취약점 스캔으로 의존성 취약점을 CI/CD 파이프라인이 잡아낸다
2. 중요한 보안 패치가 자동 테스트를 거쳐 적용된다
3. 이상 트래픽을 실시간으로 감지해 즉시 차단한다

```yaml
# DevSecOps 파이프라인 예시
stages:
  - vulnerability_scan:
      tools: [OWASP Dependency Check, Snyk]
      fail_on: [high, critical]
  - security_test:
      tools: [OWASP ZAP, Burp Suite]
  - deploy:
      condition: all_security_tests_passed
```

### 사례 2 — GitHub의 DevSecOps 성공 사례

도전 과제:
- 일일 수천 개의 코드 변경
- 수백만 개의 레포지토리 보안 관리
- 개발 속도와 보안의 균형

해결 방안:
```bash
# 1. 자동화된 보안 스캔
- CodeQL: 코드 품질 및 보안 분석
- Dependabot: 의존성 취약점 자동 업데이트
- Secret scanning: 하드코딩된 비밀정보 탐지

# 2. 개발자 친화적 보안 도구
- IDE 통합: VS Code 확장으로 실시간 보안 피드백
- PR 시 자동 보안 검토
- 교육용 Security Lab 운영
```

결과:
- 보안 취약점 발견 시간: 30일 → 1일
- 취약점 수정 시간: 14일 → 3일
- 개발자 만족도: 40% 향상

### 사례 3 — Netflix의 Chaos Engineering + Security

Netflix는 DevSecOps에 Chaos Engineering 개념을 도입했습니다.

Chaos Monkey for Security:
```python
# 예시: 자동화된 보안 테스트
import random
import time

class SecurityChaosMonkey:
    def __init__(self):
        self.attack_scenarios = [
            self.sql_injection_test,
            self.xss_test,
            self.auth_bypass_test,
            self.rate_limit_test
        ]
    
    def run_chaos(self):
        scenario = random.choice(self.attack_scenarios)
        scenario()
    
    def sql_injection_test(self):
        # 랜덤 SQL 인젝션 페이로드 테스트
        payload = "'; DROP TABLE users; --"
        self.send_request(payload)
        
    def monitor_and_alert(self):
        # 실시간 보안 이벤트 모니터링
        if self.detect_vulnerability():
            self.auto_remediation()
```

핵심 아이디어:
- 끊김 없는 보안 테스트: 프로덕션 환경에서도 안전한 보안 테스트 수행
- 자동 복구: 보안 이슈 탐지 시 즉시 자동 대응
- 학습과 개선: 매번 새로운 공격 시나리오 추가

---

## 컨테이너화 기술 심화 이해

### 왜 컨테이너인가?

#### 전통적인 배포 방식의 문제점

1. 환경 불일치 문제
```bash
# 개발자의 컴퓨터
OS: macOS Big Sur
Python: 3.9.1
Node.js: 14.15.4
Database: SQLite 3.34.0

# 프로덕션 서버
OS: Ubuntu 20.04 LTS  
Python: 3.8.5
Node.js: 12.22.0
Database: PostgreSQL 12.5

# 결과: "내 컴퓨터에서는 되는데..." 현상
```

2. 의존성 지옥 (Dependency Hell)
```bash
# 프로젝트 A
Django==3.2.0 → requires Python>=3.6

# 프로젝트 B  
TensorFlow==2.4.0 → requires Python>=3.7, <3.9

# 프로젝트 C
Some-legacy-library==1.0 → requires Python==3.6 exactly

# 한 서버에서 모든 프로젝트를 실행하는 것은 불가능
```

3. 서버 리소스 비효율성
```
전통적인 방식:
[물리 서버] → [OS] → [App A] (CPU 10% 사용)
[물리 서버] → [OS] → [App B] (CPU 15% 사용)  
[물리 서버] → [OS] → [App C] (CPU 8% 사용)

총 리소스 사용률: ~30% (70% 낭비)
```

### 컨테이너화의 핵심 개념

#### 컨테이너 vs 가상머신

```
가상머신 (Virtual Machine):
┌─────────────────────────────────────┐
│               App A                 │
├─────────────────────────────────────┤
│               Guest OS              │ ← 각 VM마다 전체 OS 필요
├─────────────────────────────────────┤
│              Hypervisor             │
├─────────────────────────────────────┤
│               Host OS               │
└─────────────────────────────────────┘

컨테이너 (Container):
┌─────────────────────────────────────┐
│    App A   │   App B   │   App C    │
├─────────────────────────────────────┤
│           Container Engine          │
├─────────────────────────────────────┤
│               Host OS               │ ← OS 공유
└─────────────────────────────────────┘
```

차이점 비교:

| 측면 | 가상머신 | 컨테이너 |
|------|----------|----------|
| 부팅 시간 | 1-2분 | 1-2초 |
| 메모리 사용량 | GB 단위 | MB 단위 |
| 격리 수준 | 완전 격리 | 프로세스 격리 |
| 포터빌리티 | 낮음 | 높음 |
| 성능 오버헤드 | 10-20% | 1-3% |

#### Linux 네임스페이스와 cgroups

컨테이너의 격리는 Linux 커널의 두 가지 기능으로 구현됩니다.

1. 네임스페이스 — 격리
```bash
# PID 네임스페이스 - 프로세스 ID 격리
컨테이너 A에서: ps aux → PID 1: nginx
컨테이너 B에서: ps aux → PID 1: apache
호스트에서: ps aux → 실제로는 다른 PID들

# Network 네임스페이스 - 네트워크 스택 격리
컨테이너 A: 10.0.1.10:80
컨테이너 B: 10.0.1.11:80  
호스트: 192.168.1.100:80

# Mount 네임스페이스 - 파일시스템 격리
컨테이너 A: /app/data → 호스트의 /var/lib/container1/data
컨테이너 B: /app/data → 호스트의 /var/lib/container2/data
```

2. cgroups (Control Groups) — 리소스 제한
```bash
# CPU 제한
echo "50000" > /sys/fs/cgroup/cpu/my-container/cpu.cfs_quota_us
# → 이 컨테이너는 CPU의 50%만 사용 가능

# 메모리 제한  
echo "512M" > /sys/fs/cgroup/memory/my-container/memory.limit_in_bytes
# → 이 컨테이너는 최대 512MB 메모리만 사용 가능

# I/O 제한
echo "1048576" > /sys/fs/cgroup/blkio/my-container/blkio.throttle.read_bps_device
# → 디스크 읽기 속도를 1MB/s로 제한
```

### Docker 아키텍처 심화

#### Docker의 내부 구조

```
Docker Client (docker CLI)
         ↓ (REST API)
Docker Daemon (dockerd)
         ↓
containerd (container runtime)
         ↓
runc (OCI runtime)
         ↓
Linux Kernel (namespaces + cgroups)
```

각 컴포넌트의 역할:

1. Docker Client
```bash
# 사용자가 입력하는 명령어들
docker build -t myapp .
docker run -p 80:80 myapp
docker ps
docker logs container_id

# 이 명령어들이 Docker Daemon에게 REST API로 전달됨
```

2. Docker Daemon (dockerd)
```bash
# 주요 책임
- 이미지 관리 (빌드, 풀, 푸시)
- 컨테이너 라이프사이클 관리
- 네트워크 및 볼륨 관리
- REST API 서버 역할
```

3. containerd
```bash
# Docker 1.11부터 분리된 컨테이너 런타임
- 컨테이너 실행 및 감독
- 이미지 전송 및 저장
- 컨테이너 네트워킹
- Kubernetes에서도 사용됨
```

4. runc
```bash
# OCI (Open Container Initiative) 표준 구현
- 실제 컨테이너 프로세스 생성
- Linux 네임스페이스 및 cgroups 설정
- 가장 저수준의 컨테이너 런타임
```

#### Dockerfile 심화 — 최적화 기법

비효율적인 Dockerfile:
```dockerfile
FROM ubuntu:20.04

# 각 RUN 명령어마다 새로운 레이어 생성 (비효율)
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y python3-pip
RUN apt-get install -y git
RUN apt-get install -y curl

COPY requirements.txt /app/
RUN pip install -r /app/requirements.txt

COPY . /app/
WORKDIR /app

CMD ["python3", "app.py"]
```

최적화된 Dockerfile:
```dockerfile
# 1. 더 작은 베이스 이미지 사용
FROM python:3.9-slim

# 2. 의존성 설치와 클린업을 한 번에 (레이어 최소화)
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        git \
        curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 3. 요구사항만 먼저 복사 (Docker 캐시 활용)
COPY requirements.txt /app/requirements.txt
WORKDIR /app
RUN pip install --no-cache-dir -r requirements.txt

# 4. 소스코드는 마지막에 복사
COPY . /app/

# 5. 비root 사용자로 실행 (보안)
RUN useradd -m appuser
USER appuser

CMD ["python3", "app.py"]
```

최적화 효과:
- 이미지 크기: 1.2GB → 150MB
- 빌드 시간: 5분 → 30초 (캐시 활용 시)
- 보안 위험도: 감소 (비root 실행)

#### 멀티스테이지 빌드

Java 애플리케이션을 예로 한 멀티스테이지 빌드:

```dockerfile
# Stage 1: Build Environment
FROM maven:3.8.1-openjdk-11-slim AS builder

WORKDIR /app
COPY pom.xml .
# 의존성만 먼저 다운로드 (캐시 최적화)
RUN mvn dependency:go-offline -B

COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Runtime Environment  
FROM openjdk:11-jre-slim

# 보안: 비root 사용자 생성
RUN useradd -r -s /bin/false appuser

WORKDIR /app

# 첫 번째 스테이지에서 빌드된 JAR만 복사
COPY --from=builder /app/target/*.jar app.jar

# 보안: 파일 권한 설정
RUN chown appuser:appuser app.jar
USER appuser

# 애플리케이션 실행
CMD ["java", "-jar", "app.jar"]
```

멀티스테이지 빌드의 장점:
1. 최종 이미지 크기 감소 — 빌드 도구들이 최종 이미지에 들어가지 않는다
2. 보안 향상 — 소스 코드와 빌드 도구가 프로덕션 이미지에 없다
3. 캐시 효율 — 스테이지마다 독립적인 캐시 레이어를 가진다

---

## 실습: DevSecOps 연구실 구축하기

자. 이론은 충분합니다. 이제 손을 더럽힐 시간입니다.
우리의 모든 실습은 Docker 위에서 진행됩니다. 여러분의 컴퓨터를 더럽히지 않으면서도 격리된 환경을 제공하기 때문이죠.

### 환경 요구사항 및 사전 체크

#### 시스템 요구사항
```bash
# 최소 사양
CPU: 2 코어 이상
RAM: 8GB 이상 (권장: 16GB)
디스크: 20GB 여유 공간
OS: Windows 10/11, macOS 10.14+, Ubuntu 18.04+

# 권장 사양
CPU: 4 코어 이상
RAM: 16GB 이상
디스크: 50GB SSD 여유 공간
네트워크: 안정적인 인터넷 연결 (패키지 다운로드용)
```

#### 사전 체크리스트

Windows 사용자:
```bash
# 1. WSL2 활성화 확인
wsl --list --verbose
# Version이 2인지 확인

# 2. Hyper-V 활성화 확인 (Pro/Enterprise만)
# Windows 기능에서 Hyper-V 체크

# 3. BIOS/UEFI에서 가상화 기술 활성화
# Intel: VT-x
# AMD: AMD-V
```

macOS 사용자:
```bash
# 1. 하드웨어 가상화 지원 확인
sysctl kern.hv_support
# kern.hv_support: 1 이어야 함

# 2. Xcode Command Line Tools 설치
xcode-select --install
```

Linux 사용자:
```bash
# 1. 커널 버전 확인 (3.10 이상 필요)
uname -r

# 2. CPU 가상화 지원 확인
grep -E "(vmx|svm)" /proc/cpuinfo
# 결과가 나와야 함
```

### Step 1 — Docker Desktop 설치 및 설정

#### Windows에서 Docker Desktop 설치

```powershell
# 1. PowerShell을 관리자 권한으로 실행
# 2. Chocolatey를 통한 설치 (선택사항)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# 3. Docker Desktop 설치
choco install docker-desktop

# 또는 직접 다운로드
# https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe
```

설치 후 설정:
```bash
# Docker Desktop 설정 → Resources → Advanced
CPU: 4개 (가능한 최대)
Memory: 8GB (전체 메모리의 50-70%)
Swap: 2GB
Disk image size: 60GB

# Kubernetes 활성화 (선택사항)
Settings → Kubernetes → Enable Kubernetes
```

#### macOS에서 Docker Desktop 설치

```bash
# 1. Homebrew를 통한 설치
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install --cask docker

# 2. 직접 다운로드
# Intel Mac: https://desktop.docker.com/mac/main/amd64/Docker.dmg
# Apple Silicon: https://desktop.docker.com/mac/main/arm64/Docker.dmg
```

#### Linux (Ubuntu)에서 Docker 설치

```bash
# 1. 기존 Docker 제거
sudo apt-get remove docker docker-engine docker.io containerd runc

# 2. 필요한 패키지 설치
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# 3. Docker GPG 키 추가
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 4. Docker 저장소 추가
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5. Docker Engine 설치
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 6. 현재 사용자를 docker 그룹에 추가
sudo usermod -aG docker $USER
newgrp docker

# 7. Docker 서비스 시작
sudo systemctl start docker
sudo systemctl enable docker
```

#### 설치 검증

```bash
# Docker 버전 확인
docker --version
# Docker version 24.0.x, build ...

# Docker Compose 확인  
docker compose version
# Docker Compose version v2.x.x

# Docker 정보 확인
docker info
# 컨테이너 개수, 이미지 개수, 스토리지 드라이버 등 정보 출력

# 테스트 실행
docker run hello-world
# Hello from Docker! 메시지가 출력되면 성공
```

### Step 2 — Jenkins 서버 구축하기

#### Jenkins 컨테이너 실행

우리는 Jenkins를 로컬에 직접 설치하지 않습니다. Docker로 단 한 줄의 명령어면 끝납니다. 이것이 현대적인 방식입니다.

```bash
# Docker 볼륨 생성 (데이터 영구 저장)
docker volume create jenkins_home

# Jenkins 컨테이너 실행
docker run -d \
  --name jenkins-master \
  --restart=unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --group-add $(getent group docker | cut -d: -f3) \
  jenkins/jenkins:lts-jdk17
```

명령어 상세 분석:

```bash
docker run                              # 컨테이너 실행
  -d                                    # 백그라운드에서 실행 (Detached mode)
  --name jenkins-master                 # 컨테이너 이름 지정
  --restart=unless-stopped              # 시스템 재부팅 시 자동 시작
  -p 8080:8080                          # 웹 UI 포트 매핑
  -p 50000:50000                        # 에이전트 연결 포트 매핑
  -v jenkins_home:/var/jenkins_home     # 데이터 볼륨 마운트
  -v /var/run/docker.sock:/var/run/docker.sock  # Docker in Docker
  --group-add $(getent group docker | cut -d: -f3)  # Docker 그룹 권한
  jenkins/jenkins:lts-jdk17             # 사용할 이미지
```

Windows 사용자는 다음 명령어 사용:
```powershell
docker run -d `
  --name jenkins-master `
  --restart=unless-stopped `
  -p 8080:8080 `
  -p 50000:50000 `
  -v jenkins_home:/var/jenkins_home `
  -v /var/run/docker.sock:/var/run/docker.sock `
  jenkins/jenkins:lts-jdk17
```

#### Jenkins 컨테이너 상태 확인

```bash
# 컨테이너 실행 확인
docker ps
# jenkins-master 컨테이너가 "Up" 상태인지 확인

# Jenkins 로그 확인  
docker logs jenkins-master
# 초기화 진행 상황을 실시간으로 볼 수 있음

# 특정 로그만 필터링
docker logs jenkins-master | grep "Jenkins initial setup"
```

### Step 3 — Jenkins 초기 설정 마법사

#### 1단계 — 초기 비밀번호 확인

브라우저에서 `http://localhost:8080` 접속하면 Unlock Jenkins 화면이 나타납니다.

```bash
# 방법 1: Docker exec으로 컨테이너 안에서 확인
docker exec jenkins-master cat /var/jenkins_home/secrets/initialAdminPassword

# 방법 2: 로그에서 찾기
docker logs jenkins-master | grep -A 5 -B 5 "Administrator password"

# 출력 예시:
# *************************************************************
# *************************************************************
# *************************************************************
# 
# Jenkins initial setup is required. An admin user has been created and a password generated.
# Please use the following password to proceed to installation:
# 
# a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
# 
# This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
```

#### 2단계 — 플러그인 설치

Install suggested plugins 선택을 권장합니다. 기본 플러그인이 DevSecOps에 필요한 기능 대부분을 커버합니다.

설치되는 주요 플러그인들:
```text
Build 관련:
- Git plugin
- GitHub plugin  
- Pipeline plugin
- Blue Ocean plugin

테스트 관련:
- JUnit plugin
- Test Results Analyzer plugin

보안 관련:
- Role-based Authorization Strategy
- LDAP plugin

배포 관련:
- SSH plugin
- Docker plugin
- Kubernetes plugin

유틸리티:
- Workspace Cleanup plugin
- Timestamper plugin
- Build Name Setter plugin
```

플러그인 설치 모니터링:
```bash
# Jenkins 로그로 설치 진행 상황 확인
docker logs -f jenkins-master

# 설치 완료 메시지를 기다림
# "Jenkins is fully up and running" 메시지 확인
```

#### 3단계 — 관리자 계정 생성

```text
Username: admin
Password: [강력한 비밀번호 설정]
Confirm password: [동일한 비밀번호]
Full name: DevSecOps Admin
E-mail address: admin@company.com
```

보안을 강화하는 비밀번호 정책:
- 최소 12자 이상
- 대소문자·숫자·특수문자 조합
- 개인정보와 관련 없는 내용
- 정기적 변경 (3개월마다)

#### 4단계 — Jenkins URL 설정

```text
Jenkins URL: http://localhost:8080/

# 실제 서버에서 운영할 때는:
# http://jenkins.company.com/
# 또는
# https://jenkins.company.com/ (SSL 적용)
```

### Step 4 — Jenkins 기본 보안 설정

#### Global Security 설정

위치: Manage Jenkins → Configure Global Security

```text
Authentication:
☑ Jenkins' own user database
☑ Allow users to sign up (개발 환경에서만)

Authorization:
☑ Logged-in users can do anything  
☑ Allow anonymous read access (선택사항)

CSRF Protection:
☑ Enable CSRF Protection (필수)

SSH Server:
Port: 50022 (기본값 또는 사용자 정의)
```

#### 사용자 및 권한 관리

```bash
# 새 사용자 생성: Manage Jenkins → Manage Users → Create User
Username: developer1
Password: [강력한 비밀번호]
Full name: Developer One
E-mail: developer1@company.com

# 권한 매트릭스 설정 (Role-based Authorization Strategy 사용 시)
Role: Developer
Permissions:
- Overall: Read
- Job: Build, Cancel, Read, Workspace
- View: Read
```

### Step 5 — 첫 번째 Jenkins Job 생성

#### Freestyle Project 생성

```text
1. Jenkins 메인 화면 → "New Item" 클릭
2. Item name: "Hello-DevSecOps"
3. "Freestyle project" 선택 → OK

설정:
General:
☑ GitHub project
Project url: https://github.com/your-username/hello-world (없으면 생략)

Source Code Management:
● None (일단 간단히 시작)

Build Triggers:
☑ Build periodically
Schedule: H/15 * * * * (15분마다 실행)

Build Environment:
☑ Delete workspace before build starts
☑ Add timestamps to the Console Output

Build Steps:
Add build step → Execute shell
```

#### Build Script 작성

```bash
#!/bin/bash
echo "=== DevSecOps 첫 번째 빌드 시작 ==="
echo "빌드 시간: $(date)"
echo "Jenkins 버전: $JENKINS_VERSION"
echo "빌드 번호: $BUILD_NUMBER"
echo "작업 공간: $WORKSPACE"

echo ""
echo "=== 시스템 정보 확인 ==="
echo "OS: $(uname -a)"
echo "CPU: $(nproc) cores"
echo "메모리: $(free -h | grep 메모리 | awk '{print $2}')"
echo "디스크: $(df -h / | tail -1 | awk '{print $4}') 여유공간"

echo ""
echo "=== 환경 변수 확인 ==="
echo "JAVA_HOME: $JAVA_HOME"  
echo "PATH: $PATH"

echo ""
echo "=== 간단한 보안 체크 ==="
# 기본 포트 스캔
if command -v netstat >/dev/null 2>&1; then
    echo "열린 포트 확인:"
    netstat -tlnp 2>/dev/null | grep LISTEN | head -5
else
    echo "netstat 명령어를 찾을 수 없음"
fi

echo ""
echo "=== 성공적으로 완료 ==="
echo "축하합니다. 첫 번째 DevSecOps 빌드가 성공했습니다."
```

#### 빌드 실행 및 확인

```text
1. "Save" 클릭하여 Job 저장
2. "Build Now" 클릭하여 즉시 실행
3. Build History에서 "#1" 클릭
4. "Console Output" 클릭하여 로그 확인

성공 시 파란색 구슬, 실패 시 빨간색 구슬 표시
```

---

## 문제 해결 및 트러블슈팅

### 자주 발생하는 문제와 해결책

#### 1. Docker 설치 관련 문제

문제: "docker: command not found"
```bash
# 해결 방법 1: PATH 확인
echo $PATH | grep docker
# docker 경로가 없으면 추가

# 해결 방법 2: 재설치 (Linux)
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# 해결 방법 3: 서비스 시작 (Linux)
sudo systemctl start docker
sudo systemctl enable docker
```

문제: "permission denied while trying to connect to the Docker daemon socket"
```bash
# 해결 방법: 사용자를 docker 그룹에 추가
sudo usermod -aG docker $USER
newgrp docker

# 또는 로그아웃 후 다시 로그인
```

문제: Windows에서 "Hardware assisted virtualization and data execution protection must be enabled"
```text
해결 방법:
1. BIOS/UEFI 진입 (부팅 시 F2/F12/DEL 키)
2. Advanced Settings 또는 CPU Configuration 메뉴 찾기
3. Intel VT-x 또는 AMD-V 활성화
4. Save & Exit
5. Windows에서 Hyper-V 기능 활성화
```

#### 2. Jenkins 접속 관련 문제

문제: "Unable to connect to Jenkins"
```bash
# 해결 방법 1: 컨테이너 상태 확인
docker ps -a | grep jenkins
# STATUS가 "Up"인지 확인

# 해결 방법 2: 포트 충돌 확인
netstat -tlnp | grep 8080
# 다른 프로세스가 8080 포트를 사용 중인지 확인

# 해결 방법 3: 다른 포트 사용
docker stop jenkins-master
docker rm jenkins-master
docker run -d --name jenkins-master -p 8081:8080 -p 50001:50000 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts-jdk17
# 브라우저에서 http://localhost:8081 접속
```

문제: 초기 비밀번호를 찾을 수 없음
```bash
# 해결 방법 1: 직접 컨테이너 안에서 확인
docker exec -it jenkins-master /bin/bash
cat /var/jenkins_home/secrets/initialAdminPassword
exit

# 해결 방법 2: 로그에서 더 자세히 검색
docker logs jenkins-master | grep -i password

# 해결 방법 3: 컨테이너 재시작
docker restart jenkins-master
docker logs -f jenkins-master
```

#### 3. 빌드 실행 관련 문제

문제: "Build failed with exit code 127"
```text
원인: 명령어를 찾을 수 없음 (command not found)

해결 방법:
1. 명령어 경로 확인: which [명령어]
2. 필요한 도구 설치:
   - apt-get update && apt-get install -y [패키지명]
   - yum install -y [패키지명]
3. PATH 환경변수에 경로 추가
```

문제: "Permission denied" 에러
```bash
# 해결 방법 1: 파일 권한 확인
ls -la /path/to/file
# 실행 권한이 없으면 추가
chmod +x /path/to/script.sh

# 해결 방법 2: Jenkins 사용자 권한 확인
docker exec jenkins-master whoami
# jenkins 사용자로 실행되는지 확인
```

#### 4. 메모리 및 성능 문제

문제: Jenkins가 느리거나 메모리 부족
```bash
# 해결 방법 1: Java 힙 메모리 증가
docker stop jenkins-master
docker rm jenkins-master
docker run -d \
  --name jenkins-master \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -e JAVA_OPTS="-Xmx2g -Xms1g" \
  jenkins/jenkins:lts-jdk17

# 해결 방법 2: 불필요한 플러그인 제거
# Manage Jenkins → Manage Plugins → Installed 탭에서 사용하지 않는 플러그인 제거

# 해결 방법 3: 빌드 기록 정리
# Job 설정 → Build Triggers → "Discard old builds" 체크
# Keep builds for: 30 days
# Max # of builds to keep: 50
```

#### 5. 네트워크 관련 문제

문제: 외부 저장소에 접속할 수 없음
```bash
# 해결 방법 1: DNS 확인
docker exec jenkins-master nslookup github.com
# DNS 해결이 안 되면 네트워크 설정 확인

# 해결 방법 2: 프록시 설정
# Manage Jenkins → Manage Plugins → Advanced
# HTTP Proxy Configuration에 프록시 서버 정보 입력

# 해결 방법 3: 방화벽 확인
# 필요한 포트들이 열려있는지 확인
# 80/443 (HTTP/HTTPS), 22 (SSH), 9418 (Git)
```

### 로그 분석 및 디버깅 기법

#### Jenkins 로그 레벨 설정

```text
위치: Manage Jenkins → System Log → All Jenkins Logs

로그 레벨 추가:
Logger: hudson.model.Run
Level: FINE (상세한 빌드 로그)

Logger: jenkins.security  
Level: FINE (보안 관련 로그)

Logger: hudson.plugins.git
Level: FINE (Git 관련 로그)
```

#### 유용한 로그 명령어

```bash
# 실시간 로그 모니터링
docker logs -f jenkins-master

# 최근 100줄만 확인
docker logs --tail 100 jenkins-master

# 특정 시간 이후 로그만 확인  
docker logs --since="2024-01-30T10:00:00" jenkins-master

# 로그를 파일로 저장
docker logs jenkins-master > jenkins.log 2>&1
```

---

## 1주차 심화 과제

### 기본 과제 (필수)

#### 과제 1 — Jenkins 마스터 환경 구축
```text
목표: 완전한 Jenkins 환경을 구축하고 기본 설정을 완료한다.

요구사항:
1. Docker를 사용하여 Jenkins 컨테이너 실행
2. 관리자 계정 생성 및 기본 보안 설정 적용
3. 최소 5개 이상의 필수 플러그인 설치
4. 시스템 정보를 출력하는 기본 Job 생성

제출물:
- Jenkins 대시보드 스크린샷
- 첫 번째 빌드 성공 화면 스크린샷  
- 설치된 플러그인 목록 (텍스트 파일)
- 학습 일지 (500자 이상)
```

#### 과제 2 — Docker 환경 최적화
```bash
# 목표: Docker 환경을 최적화하고 모니터링 도구를 설정한다.

# 1. Docker 시스템 정보 수집
docker system info > docker-info.txt
docker system df > docker-usage.txt

# 2. 불필요한 리소스 정리
docker system prune -a --volumes

# 3. Docker 모니터링 대시보드 구축
docker run -d \
  --name=portainer \
  -p 9000:9000 \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce

# 4. cAdvisor로 컨테이너 모니터링
docker run -d \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8082:8080 \
  --detach=true \
  --name=cadvisor \
  gcr.io/cadvisor/cadvisor:latest
```

### 중급 과제 (선택)

#### 과제 3 — 멀티 컨테이너 환경 구축
Docker Compose로 Jenkins + 데이터베이스 + 리버스 프록시 환경을 구축합니다.

```yaml
# docker-compose.yml
version: '3.8'
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - jenkins
    restart: unless-stopped

  jenkins:
    image: jenkins/jenkins:lts-jdk17
    ports:
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JAVA_OPTS=-Xmx2g -Xms1g
    restart: unless-stopped

  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: jenkins
      POSTGRES_USER: jenkins
      POSTGRES_PASSWORD: secure_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:alpine
    restart: unless-stopped

volumes:
  jenkins_home:
  postgres_data:
```

#### 과제 4 — 기본 보안 스캔 파이프라인 구축
```groovy
// Jenkinsfile
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your-username/sample-app.git'
            }
        }
        
        stage('Security Scan') {
            parallel {
                stage('Secret Scan') {
                    steps {
                        sh '''
                            # TruffleHog으로 시크릿 스캔
                            docker run --rm -v "$PWD:/pwd" \
                                trufflesecurity/trufflehog:latest \
                                filesystem /pwd --json > secrets-report.json
                        '''
                        archiveArtifacts 'secrets-report.json'
                    }
                }
                
                stage('Dependency Scan') {
                    steps {
                        sh '''
                            # OWASP Dependency Check
                            docker run --rm \
                                -v "$PWD:/src" \
                                -v "$HOME/.gradle:/root/.gradle" \
                                owasp/dependency-check:latest \
                                --scan /src \
                                --format JSON \
                                --out /src
                        '''
                        archiveArtifacts 'dependency-check-report.json'
                    }
                }
            }
        }
        
        stage('Report') {
            steps {
                sh '''
                    echo "=== 보안 스캔 결과 요약 ==="
                    echo "1. 시크릿 검사 완료"
                    echo "2. 의존성 취약점 검사 완료"
                    echo "상세 결과는 아티팩트를 확인하세요."
                '''
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

### 고급 과제 (도전)

#### 과제 5 — CI/CD 성능 벤치마킹
```python
#!/usr/bin/env python3
# performance_benchmark.py

import time
import requests
import subprocess
import json
import statistics

def measure_build_time(job_name, num_runs=5):
    """Jenkins Job의 빌드 시간을 측정"""
    jenkins_url = "http://localhost:8080"
    auth = ('admin', 'your_password')  # 실제 인증정보로 변경
    
    build_times = []
    
    for i in range(num_runs):
        print(f"빌드 {i+1}/{num_runs} 시작...")
        
        # 빌드 시작
        start_time = time.time()
        response = requests.post(
            f"{jenkins_url}/job/{job_name}/build",
            auth=auth
        )
        
        if response.status_code == 201:
            # 빌드 완료 대기
            while True:
                status_response = requests.get(
                    f"{jenkins_url}/job/{job_name}/lastBuild/api/json",
                    auth=auth
                )
                
                if status_response.status_code == 200:
                    build_info = status_response.json()
                    if not build_info['building']:
                        end_time = time.time()
                        build_time = end_time - start_time
                        build_times.append(build_time)
                        print(f"빌드 {i+1} 완료: {build_time:.2f}초")
                        break
                
                time.sleep(2)
        else:
            print(f"빌드 시작 실패: {response.status_code}")
    
    # 통계 계산
    if build_times:
        avg_time = statistics.mean(build_times)
        min_time = min(build_times)
        max_time = max(build_times)
        std_dev = statistics.stdev(build_times) if len(build_times) > 1 else 0
        
        print(f"\n=== 빌드 성능 분석 ===")
        print(f"평균 빌드 시간: {avg_time:.2f}초")
        print(f"최소 빌드 시간: {min_time:.2f}초")
        print(f"최대 빌드 시간: {max_time:.2f}초")
        print(f"표준 편차: {std_dev:.2f}초")
        
        return {
            'job_name': job_name,
            'build_times': build_times,
            'avg_time': avg_time,
            'min_time': min_time,
            'max_time': max_time,
            'std_dev': std_dev
        }

def measure_resource_usage():
    """Docker 컨테이너 리소스 사용량 측정"""
    result = subprocess.run([
        'docker', 'stats', '--no-stream', '--format',
        'table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}'
    ], capture_output=True, text=True)
    
    print("\n=== 리소스 사용량 ===")
    print(result.stdout)

if __name__ == "__main__":
    # 벤치마킹 실행
    results = measure_build_time("Hello-DevSecOps", 3)
    measure_resource_usage()
    
    # 결과를 JSON으로 저장
    with open('benchmark_results.json', 'w') as f:
        json.dump(results, f, indent=2)
    
    print("\n결과가 benchmark_results.json에 저장되었습니다.")
```

---

## 자가평가 및 다음 주 예고

### 1주차 체크리스트

기본 개념 이해:
- [ ] DevOps와 DevSecOps의 차이점을 설명한다
- [ ] Shift Left 보안의 의미와 장점을 안다
- [ ] 컨테이너와 가상머신의 차이를 구분한다
- [ ] Docker의 기본 아키텍처를 설명한다

실습 완료:
- [ ] Docker Desktop을 성공적으로 설치했다
- [ ] Jenkins 컨테이너를 실행하고 접속했다
- [ ] 첫 번째 Jenkins Job을 생성하고 실행했다
- [ ] 기본적인 문제 해결을 수행한다

심화 학습:
- [ ] Dockerfile을 작성하고 최적화한다
- [ ] Docker Compose를 활용한다
- [ ] 기본 보안 스캔 도구를 다룬다

### Self-Assessment Quiz

#### 퀴즈 1 — 기본 개념 (각 10점, 총 50점)

Q1.1 DevOps의 CALMS 모델 중 "A"는 무엇을 의미하나요?
- A) Architecture (아키텍처)
- B) Automation (자동화)
- C) Agility (민첩성)
- D) Analytics (분석)

정답: B
해설: CALMS는 Culture, Automation, Lean, Measurement, Sharing을 의미합니다.

---

Q1.2 Shift Left 보안이 해결하려는 주요 문제는?
- A) 개발 속도가 느린 문제
- B) 서버 비용이 비싼 문제
- C) 보안 취약점을 늦게 발견하는 문제
- D) 코드 품질이 낮은 문제

정답: C
해설: Shift Left는 보안 검토를 개발 초기 단계로 앞당겨 취약점을 조기에 발견하고 수정 비용을 줄이려는 접근입니다.

---

Q1.3 Docker 컨테이너의 격리를 구현하는 Linux 커널 기능은?
- A) systemd와 cgroups
- B) namespaces와 cgroups
- C) LXC와 systemd
- D) chroot와 iptables

정답: B
해설: Docker는 Linux namespaces(격리)와 cgroups(리소스 제한)로 컨테이너 격리를 구현합니다.

---

Q1.4 다음 중 멀티스테이지 Docker 빌드의 주요 장점이 아닌 것은?
- A) 최종 이미지 크기 감소
- B) 빌드 속도 향상
- C) 보안 향상 (소스코드 제외)
- D) 개발/프로덕션 환경 분리

정답: B
해설: 멀티스테이지 빌드는 이미지 크기 감소와 보안 향상에는 도움이 되지만 빌드 속도는 오히려 더 걸리기도 합니다.

---

Q1.5 Jenkins에서 "Build periodically" 설정의 "H/15 * * * *"는 무엇을 의미하나요?
- A) 매시 15분에 실행
- B) 15분마다 실행
- C) 매일 15시에 실행
- D) 매월 15일에 실행

정답: B
해설: H/15는 0~59분 사이에서 15분 간격으로 실행을 의미합니다. H는 부하 분산용 해시값입니다.

#### 퀴즈 2 — 실습 응용 (각 10점, 총 30점)

Q2.1 Jenkins 컨테이너에서 Docker 명령어를 쓰려면 어떤 설정이 필요한가요?
- A) --privileged 플래그만 추가
- B) Docker 이미지를 별도로 설치
- C) /var/run/docker.sock을 마운트
- D) 네트워크 모드를 host로 설정

정답: C
해설: Docker in Docker를 하려면 호스트의 Docker 소켓을 컨테이너에 마운트해야 합니다.

---

Q2.2 Jenkins Job이 "Exit code 127"로 실패하는 가장 일반적인 원인은?
- A) 메모리 부족
- B) 디스크 공간 부족
- C) 명령어를 찾을 수 없음 (command not found)
- D) 권한 부족

정답: C
해설: Exit code 127은 Unix/Linux에서 명령어를 못 찾을 때 반환하는 표준 종료 코드입니다.

---

Q2.3 다음 중 Jenkins 성능 최적화 방법이 아닌 것은?
- A) Java 힙 메모리 증가
- B) 빌드 기록 정리 설정
- C) 불필요한 플러그인 제거
- D) 모든 빌드를 마스터 노드에서 실행

정답: D
해설: 성능을 끌어올리려면 빌드 작업을 에이전트 노드로 분산해야 하며, 마스터 노드 하나에서 모든 빌드를 돌리는 것은 비효율적입니다.

#### 퀴즈 3 — 시나리오 분석 (20점)

Q3.1 다음 시나리오에서 가장 적절한 해결 방법은?

시나리오: 개발팀이 매주 금요일마다 손으로 서버에 배포하고 있습니다. 배포 과정에서 자주 문제가 터지고, 문제 발생 시 원인 파악이 어려워 복구에 평균 4시간이 듭니다. 보안팀에서는 배포 전 보안 검토를 요청했지만, 일정 압박 탓에 종종 생략됩니다.

다음 중 이 문제를 푸는 데 가장 종합적인 접근 방법은?

- A) 더 자세한 배포 문서를 작성하고, 보안팀 검토 시간을 단축한다
- B) 자동화된 CI/CD 파이프라인을 구축하고, 보안 검사를 파이프라인에 통합한다
- C) 배포를 매일 진행하여 문제 발생 시 영향을 줄인다
- D) 별도의 보안 전담팀을 두어 모든 배포를 검토한다

정답: B
해설: DevSecOps의 핵심은 자동화된 파이프라인에 보안을 녹여 넣는 것입니다. 반복적인 실수를 줄이고 빠른 피드백으로 문제를 일찍 잡아냅니다.

---

### 다음 주 예고 — Jenkins 아키텍처 마스터하기

Week 2에서 배울 내용:

#### Jenkins 내부 아키텍처 심화
- Master-Agent 분산 빌드 시스템: 스케일링과 부하 분산
- Jenkins 파일시스템 구조: 플러그인, Job, 빌드 데이터 저장 방식
- 보안 아키텍처: 인증, 인가, 감사 로그 시스템

#### 고급 설정과 최적화
- 플러그인 생태계 활용: 필수 플러그인 vs 선택 플러그인
- 성능 튜닝: JVM 옵션, 가비지 컬렉션 최적화
- 백업과 복원: 재해 복구를 위한 Jenkins 데이터 관리

#### 엔터프라이즈 보안 설정
- LDAP/AD 통합: 조직의 사용자 디렉터리와 연동
- Role-based Access Control: 세밀한 권한 관리
- 감사 로그: 모든 활동 추적과 컴플라이언스

#### 고가용성 설정
- 클러스터 구성: 여러 마스터 노드로 고가용성 달성
- 로드 밸런싱: 트래픽 분산과 장애 조치
- 모니터링: Prometheus/Grafana로 실시간 감시

### 사전 학습 권장사항

1. Linux 기본 명령어 복습: chmod, chown, systemctl, ps, netstat
2. YAML 문법 학습: Kubernetes, Docker Compose에서 사용
3. Git 기본 명령어: clone, pull, push, merge, branch
4. 네트워킹 기초: TCP/IP, DNS, SSL/TLS 기본 개념

### 1주차 마무리 팁

성공적인 DevSecOps 여정을 위한 조언:

1. 꾸준함이 핵심입니다. 매일 30분이라도 손에서 놓지 마세요
2. 커뮤니티를 적극 활용하세요. Stack Overflow, Reddit r/DevOps, 국내 DevOps 카페 모두 좋습니다
3. 실험 정신을 잊지 마세요. 망가뜨리는 것을 두려워하지 않아도 됩니다. 컨테이너는 언제든 다시 만들면 됩니다
4. 문서화 습관을 들이세요. 배운 내용과 트러블슈팅 과정을 그때그때 기록하세요
5. 보안 마인드셋, Security by Design — 처음부터 보안을 고려하는 버릇을 들이세요

---

"DevSecOps는 도구가 아닙니다. 문화입니다."

다음 주에는 Jenkins의 심장부를 해부하고, 진짜 프로가 쓰는 고급 설정들을 마스터하겠습니다. 여러분의 DevSecOps 여정은 이제 시작입니다.

Keep Automating, Stay Secure.
