---
layout: post
title: "k9s 쿠버네티스 완벽 운영 Week 2: 쿠버네티스 아키텍처 심화 & k9s 고급 네비게이션 마스터"
date: "2026-05-19 05:56:00 +0900"
categories: [DevOps, Kubernetes]
tags: [k9s, Kubernetes, Architecture, ControlPlane, etcd, Week2]
---

환영합니다! Week 2에서는 단순히 k9s 사용법을 넘어서, 쿠버네티스의 심장부를 들여다보고 k9s로 이를 어떻게 완벽하게 제어할 수 있는지 배우겠습니다. 이번 주가 끝나면 여러분은 쿠버네티스 클러스터의 내부 동작을 완전히 이해하고, k9s를 통해 마치 클러스터의 뇌파를 읽듯 모든 상태를 파악할 수 있게 됩니다.

---

## 🎯 Week 2 학습 목표

이번 주차를 마치면 여러분은:
1. **쿠버네티스 Control Plane의 실제 동작 원리**를 시스템 수준에서 이해
2. **etcd, API Server, Scheduler의 상호작용**을 k9s로 실시간 관찰
3. **k9s 고급 네비게이션 테크닉**으로 복잡한 클러스터를 자유자재로 탐색
4. **멀티 클러스터 환경에서의 컨텍스트 관리** 마스터
5. **실제 Google GKE 운영 시나리오**를 통한 실전 경험 습득

---

## 🏗️ 1교시: 쿠버네티스 아키텍처 해부학

### 뇌수술사가 되어야 하는 이유

여러분이 응급실 의사라고 상상해보세요. 환자가 의식을 잃고 들어왔을 때, 외적 증상만 보고 치료할 수 있을까요? 뇌의 구조와 동작 원리를 모르면 불가능합니다. 쿠버네티스도 마찬가지입니다. 

### Control Plane의 4대 천왕

#### 1. API Server - "대통령의 비서실"

API Server는 모든 요청의 관문입니다. kubectl, k9s, 심지어 다른 컴포넌트들도 모두 API Server를 통해서만 소통합니다.

**실제 동작 방식:**
```bash
# k9s에서 API Server 상태 확인
:pods -n kube-system
# kube-apiserver-* Pod 찾기

# API Server 로그 확인으로 실제 요청 추적
l  # 로그 보기
# 여러분이 k9s에서 무언가 할 때마다 로그에 찍히는 것을 확인
```

**실전 디버깅 사례:**
```
증상: k9s가 갑자기 느려짐
원인 탐색:
1. k9s → :pods -n kube-system → kube-apiserver 선택
2. d (describe) → 리소스 사용률 확인
3. l (logs) → 에러 메시지나 과도한 요청 확인
4. :no (nodes) → Control Plane 노드 상태 확인
```

#### 2. etcd - "국가 기록보관소"

etcd는 클러스터의 모든 상태 정보를 저장하는 분산 키-값 저장소입니다. etcd가 죽으면 클러스터 전체가 마비됩니다.

**etcd 건강도 체크 with k9s:**
```bash
# etcd Pod 상태 확인
:pods -n kube-system
/etcd

# etcd 메트릭 확인 (k9s에서 직접)
:pods -n kube-system → etcd 선택 → d
# Events 섹션에서 이상 징후 탐지

# etcd 데이터 크기 모니터링
# Custom Resource Definition으로 etcd 메트릭 추가 설정
```

**실제 사고 사례 - 삼성전자 클라우드 팀:**
```
상황: etcd 디스크 용량 부족으로 클러스터 전체 장애
조치: k9s로 etcd Pod 리소스 사용률 실시간 모니터링
결과: 디스크 사용률 95% 도달 전 사전 확장으로 장애 예방
```

#### 3. Scheduler - "최고의 인사담당자"

Scheduler는 새로 생성된 Pod를 어느 노드에 배치할지 결정합니다. 이 결정은 리소스 효율성과 애플리케이션 성능에 직결됩니다.

**Scheduler 의사결정 과정 추적:**
```bash
# 새 Pod 생성하면서 스케줄링 과정 관찰
kubectl run test-pod --image=nginx

# k9s에서 실시간 관찰
:pods
# test-pod 상태 변화: Pending → ContainerCreating → Running

# Scheduler 로그에서 배치 결정 과정 확인
:pods -n kube-system
/scheduler
l  # 로그 확인
```

**스케줄링 실패 디버깅:**
```
문제: Pod이 Pending 상태로 멈춤
진단:
1. k9s → 해당 Pod 선택 → d (describe)
2. Events 섹션에서 스케줄링 실패 원인 확인
   - Insufficient memory
   - No nodes available
   - Node selector mismatch
3. :no (nodes) → 각 노드의 리소스 확인
4. 해결책 적용: 노드 추가 or 리소스 요청량 조정
```

#### 4. Controller Manager - "완벽주의 감독관"

Controller Manager는 클러스터의 '이상적인 상태'를 유지하는 역할을 합니다. ReplicaSet, Deployment, Service 등의 컨트롤러들을 관리합니다.

**Controller 동작 실시간 관찰:**
```bash
# Deployment 스케일링으로 Controller 동작 확인
kubectl scale deployment nginx --replicas=5

# k9s에서 실시간 관찰
:deploy → nginx 선택
# DESIRED vs CURRENT vs READY 값 변화 관찰

:pods
# 새 Pod들이 생성되는 과정 실시간 추적
```

### k9s로 Control Plane 건강도 종합 점검

```bash
# 1단계: 전체 시스템 Pod 상태
:pods -n kube-system

# 2단계: 각 컴포넌트별 리소스 사용률
d (각 컴포넌트 Pod 선택 후 describe)

# 3단계: 로그 이상 징후 탐지
l (각 컴포넌트별 로그 확인)

# 4단계: 노드 상태 확인
:no
# Ready 상태, 리소스 사용률 확인

# 5단계: 중요한 시스템 이벤트 확인
:ev -n kube-system
# 최근 이벤트로 문제 징후 파악
```

---

## 🧭 2교시: k9s 고급 네비게이션 - 프로의 기법

### 멀티 차원 클러스터 탐색

#### 네임스페이스 차원의 마스터
```bash
# 빠른 네임스페이스 전환
ctrl+a    # 전체 네임스페이스 토글
ctrl+n    # 다음 네임스페이스
ctrl+p    # 이전 네임스페이스

# 네임스페이스 검색과 필터링
:ns
/kube-system    # 특정 네임스페이스 검색
enter           # 해당 네임스페이스로 이동
```

#### 리소스 타입별 고속 이동
```bash
# 가장 많이 사용하는 단축키
shift+:  # 명령 팔레트 (모든 리소스 타입 검색 가능)

# 자주 사용하는 리소스 별칭
:po      # pods
:svc     # services  
:deploy  # deployments
:rs      # replicasets
:ds      # daemonsets
:sts     # statefulsets
:job     # jobs
:cj      # cronjobs
:pv      # persistentvolumes
:pvc     # persistentvolumeclaims
:cm      # configmaps
:sec     # secrets
:sa      # serviceaccounts
:no      # nodes
:ns      # namespaces
```

#### 고급 검색과 필터링 테크닉

**정규식 검색:**
```bash
/nginx.*prod    # nginx로 시작하고 prod가 포함된 리소스
/.*error.*      # error가 포함된 모든 리소스
/^web-          # web-으로 시작하는 리소스
```

**상태 기반 필터링:**
```bash
# Pod 상태별 필터링
/Running        # 실행 중인 Pod만
/Pending        # 대기 중인 Pod만
/Failed         # 실패한 Pod만
/CrashLoopBackOff  # 반복 실패 Pod만

# 부분 일치 검색 (대소문자 무관)
/ERROR          # error, Error, ERROR 모두 매치
```

### 컨텍스트와 클러스터 관리

#### 멀티 클러스터 환경에서의 k9s 활용
```bash
# 현재 컨텍스트 확인
:ctx

# 다른 클러스터로 빠른 전환
:ctx
# 화살표로 선택 후 enter

# 또는 k9s 시작 시 컨텍스트 지정
k9s --context production
k9s --context staging
k9s --context development
```

#### 실제 운영 환경에서의 컨텍스트 관리 전략

**구글 Cloud의 멀티 클러스터 시나리오:**
```bash
# 개발 환경 확인
k9s --context gke_project-dev_us-central1_dev-cluster

# 스테이징 환경 확인  
k9s --context gke_project-staging_us-central1_staging-cluster

# 프로덕션 환경 확인 (읽기 전용)
k9s --context gke_project-prod_us-central1_prod-cluster --readonly
```

### k9s 사용자 정의 설정으로 효율성 극대화

#### 개인화된 별칭 설정
```yaml
# ~/.k9s/alias.yml
alias:
  # 자주 사용하는 리소스 조합
  pp: v1/pods
  dp: apps/v1/deployments  
  svc: v1/services
  
  # 네임스페이스별 빠른 접근
  kube: kube-system
  prod: production
  dev: development
  
  # 복합 검색
  errors: "pods --field-selector status.phase=Failed"
  pending: "pods --field-selector status.phase=Pending"
```

#### 핫키 커스터마이징
```yaml
# ~/.k9s/hotkey.yml
hotKey:
  # 빠른 재시작
  shift+r:
    shortCut: Shift+R
    description: Rolling restart deployment
    command: rollout/restart
  
  # 빠른 스케일링  
  shift+s:
    shortCut: Shift+S
    description: Scale deployment
    command: scale
  
  # 로그 스트리밍
  shift+l:
    shortCut: Shift+L  
    description: Stream logs
    command: logs/stream
```

### 고급 로그 분석 기법

#### 실시간 로그 스트리밍과 분석
```bash
# Pod 로그 기본 보기
:pods → Pod 선택 → l

# 고급 로그 옵션
l → f     # follow (실시간 스트리밍)
l → p     # previous (이전 컨테이너 로그)
l → c     # container 선택 (멀티 컨테이너 Pod)

# 로그 검색
로그 화면에서 /keyword 로 검색
```

#### 여러 Pod 로그 동시 모니터링 전략
```bash
# 방법 1: 여러 k9s 인스턴스 실행
# 터미널 분할 후 각각 다른 Pod 로그 모니터링

# 방법 2: label selector 활용
:pods
/app=nginx    # 같은 라벨을 가진 Pod들 필터링
# 각 Pod 로그를 순차적으로 확인
```

---

## 🌐 3교시: 실전 시나리오 - Google GKE 클러스터 운영

### 시나리오 설정: 글로벌 이커머스 플랫폼

여러분은 아시아 최대 이커머스 플랫폼인 "ShopAsia"의 SRE 팀장입니다. 다음과 같은 인프라를 운영하고 있습니다:

```
Production (Seoul):      gke_shopasia-prod_asia-northeast3_prod-seoul
Staging (Tokyo):         gke_shopasia-staging_asia-northeast1_staging-tokyo  
Development (Singapore): gke_shopasia-dev_asia-southeast1_dev-singapore
```

각 클러스터는 다음 구조를 가집니다:
- **frontend**: 웹 UI 서비스들
- **backend**: API 및 마이크로서비스들  
- **database**: 데이터베이스 관련 서비스들
- **monitoring**: 모니터링 스택

### 미션 1: 새벽 3시 장애 대응

**상황:**
새벽 3시, PagerDuty 알람이 울립니다. 서울 프로덕션 클러스터의 주문 처리 서비스에서 50% 에러율을 기록하고 있습니다.

#### Step 1: 빠른 상황 파악
```bash
# 프로덕션 클러스터 연결
k9s --context gke_shopasia-prod_asia-northeast3_prod-seoul

# 전체적인 클러스터 건강도 확인
:no  # 노드 상태 확인
:ns  # 네임스페이스 개요 확인
```

#### Step 2: 문제 영역 식별
```bash
# backend 네임스페이스로 이동
:ns → /backend → enter

# 또는 직접 이동
ctrl+shift+n → backend 입력

# Pod 상태 확인
:pods
# READY 컬럼에서 이상한 비율 확인 (예: 1/2, 0/1)
# STATUS 컬럼에서 오류 상태 확인
```

#### Step 3: 문제 Pod 심층 분석
```bash
# 에러 상태 Pod 필터링
/Error
/CrashLoopBackOff
/ImagePullBackOff

# 문제 Pod 선택 후 상세 분석
d  # describe로 이벤트 확인
l  # 로그로 에러 메시지 확인
y  # YAML로 설정 확인
```

#### Step 4: 관련 서비스 체크
```bash
# 서비스 상태 확인
:svc
/order  # 주문 관련 서비스 검색

# 엔드포인트 확인
:ep (endpoints)
# 서비스가 실제로 연결된 Pod 확인
```

#### Step 5: 리소스 사용률 분석
```bash
# 노드별 리소스 압박 확인
:no
s  # 정렬 옵션으로 CPU 또는 메모리 사용률 순 정렬

# 특정 Pod의 리소스 사용률 확인
:pods → 문제 Pod 선택 → d
# Resources 섹션 확인
```

### 미션 2: 스테이징 환경에서의 배포 테스트

**상황:**
새로운 주문 처리 로직을 스테이징 환경에 배포하고 k9s로 모니터링해야 합니다.

#### Step 1: 스테이징 클러스터로 전환
```bash
k9s --context gke_shopasia-staging_asia-northeast1_staging-tokyo
```

#### Step 2: 배포 전 기준점 확인
```bash
# 현재 배포 상태 저장
:deploy -n backend
# order-service의 현재 이미지 태그, 레플리카 수 확인

:pods -n backend  
/order-service
# 현재 실행 중인 Pod 상태 스냅샷
```

#### Step 3: 배포 실행 및 실시간 모니터링
```bash
# 새 터미널에서 배포 실행
kubectl set image deployment/order-service order-service=order-service:v2.1.0 -n backend

# k9s에서 실시간 모니터링
:deploy -n backend → order-service 선택
# READY 컬럼에서 rolling update 진행 상황 확인

:rs -n backend  
/order-service
# 새로운 ReplicaSet 생성과 기존 ReplicaSet 축소 과정 관찰

:pods -n backend
/order-service  
# 새 Pod 생성, 구 Pod 종료 과정 실시간 관찰
```

#### Step 4: 배포 검증
```bash
# 새 Pod들의 로그 확인
:pods -n backend → 새 order-service Pod 선택 → l
# 애플리케이션 시작 로그에서 에러 메시지 확인

# 서비스 연결 상태 확인  
:svc -n backend → order-service 선택 → d
# Endpoints 섹션에서 새 Pod IP 연결 확인
```

### 미션 3: 개발 환경에서의 새 기능 테스트

**상황:**
개발팀이 새로운 마이크로서비스를 개발 클러스터에 배포했습니다. k9s로 통합 테스트를 지원해야 합니다.

#### Step 1: 개발 클러스터 연결
```bash
k9s --context gke_shopasia-dev_asia-southeast1_dev-singapore
```

#### Step 2: 새 서비스 확인
```bash
# 최근 배포된 리소스 찾기
:deploy -n backend
s  # 정렬을 AGE로 변경
# 가장 최근 배포 확인

:pods -n backend
s  # AGE 순 정렬로 새 Pod 식별
```

#### Step 3: 서비스 간 통신 테스트 지원
```bash
# 새 서비스의 네트워크 연결성 확인
:svc -n backend → 새 서비스 선택 → d
# ClusterIP, Ports 확인

# 다른 서비스에서의 연결 테스트 지원
:pods -n backend → 다른 서비스 Pod 선택
shift+s  # shell 접속
# 내부에서 curl 테스트 실행
curl http://new-service:8080/health
exit
```

### 미션 4: 크로스 클러스터 상태 비교

#### 컨텍스트 빠른 전환으로 환경 간 비교
```bash
# 프로덕션 환경 확인
k9s --context prod-seoul
:deploy -n backend → order-service
# 이미지 태그: v1.9.5, 레플리카: 10

# 스테이징 환경 확인  
:ctx → staging-tokyo 선택
:deploy -n backend → order-service  
# 이미지 태그: v2.1.0, 레플리카: 3

# 개발 환경 확인
:ctx → dev-singapore 선택
:deploy -n backend → order-service
# 이미지 태그: v2.2.0-alpha, 레플리카: 1
```

---

## 🎛️ 4교시: k9s 고급 기능 - 메트릭과 모니터링 통합

### Prometheus 메트릭 통합 활용

#### k9s에서 실시간 메트릭 확인
```bash
# 노드별 메트릭 확인
:no
# CPU%, MEM% 컬럼에서 실시간 사용률 확인

# Pod별 리소스 사용률
:pods  
s  # CPU 또는 Memory로 정렬
# 리소스를 가장 많이 사용하는 Pod 식별
```

#### HPA (Horizontal Pod Autoscaler) 모니터링
```bash
# HPA 상태 실시간 관찰
:hpa
# TARGETS 컬럼: 현재/타겟 메트릭 값
# REPLICAS 컬럼: 현재/최소/최대 레플리카 수
# AGE 컬럼: 마지막 스케일링 시점

# 특정 HPA 선택 후 상세 정보
d  # describe로 스케일링 이벤트 히스토리 확인
```

#### VPA (Vertical Pod Autoscaler) 권장사항 확인
```bash
# VPA 오브젝트 확인
:vpa
# RECOMMENDATION 컬럼에서 권장 리소스 확인

# VPA 대상 Pod 리소스와 권장사항 비교
:vpa → VPA 선택 → d
# 권장사항과 현재 설정 비교 분석
```

### 네트워크 정책과 보안 모니터링

#### Network Policy 적용 상태 확인
```bash
# 네트워크 정책 목록
:netpol

# 특정 정책 선택 후 상세 확인
d  # 어떤 Pod에 적용되는지, 어떤 트래픽을 허용/차단하는지
```

#### Pod Security Policy 준수 상태
```bash
# Pod 보안 정책 위반 확인
:pods
d  # describe에서 Security Context 확인
# runAsNonRoot, readOnlyRootFilesystem 등 보안 설정 점검
```

### 이벤트 기반 모니터링

#### 클러스터 전체 이벤트 분석
```bash
# 전체 이벤트 보기
:events
# TYPE 컬럼: Normal vs Warning
# REASON 컬럼: 이벤트 발생 원인  
# MESSAGE 컬럼: 상세 메시지

# 경고/오류 이벤트 필터링
/Warning
/Error
/Failed
```

#### 네임스페이스별 이벤트 분석
```bash
:events -n production
# 특정 네임스페이스의 이벤트만 필터링

# 최근 이벤트 정렬
s  # AGE 순으로 정렬해서 최신 이벤트부터 확인
```

---

## 📊 실전 프로젝트: 멀티 클러스터 대시보드 구축

### 프로젝트 목표
k9s와 스크립트를 조합해서 여러 클러스터의 상태를 종합 모니터링하는 시스템 구축

### Step 1: 클러스터별 상태 수집 스크립트
```bash
#!/bin/bash
# cluster-health-check.sh

CLUSTERS=("prod-seoul" "staging-tokyo" "dev-singapore")
NAMESPACES=("frontend" "backend" "database")

for cluster in "${CLUSTERS[@]}"; do
    echo "=== $cluster 클러스터 상태 ==="
    
    # 컨텍스트 전환
    kubectl config use-context $cluster
    
    for ns in "${NAMESPACES[@]}"; do
        echo "--- $ns 네임스페이스 ---"
        
        # Pod 상태 요약
        kubectl get pods -n $ns --no-headers | \
        awk '{print $3}' | sort | uniq -c
        
        # 리소스 사용률 상위 5개 Pod
        kubectl top pods -n $ns --sort-by=cpu | head -6
    done
    
    echo ""
done
```

### Step 2: k9s 설정 파일로 원클릭 모니터링
```yaml
# ~/.k9s/clusters.yml
clusters:
  - name: "Production Seoul"
    context: "gke_shopasia-prod_asia-northeast3_prod-seoul"
    namespaces: ["frontend", "backend", "database"]
    
  - name: "Staging Tokyo"  
    context: "gke_shopasia-staging_asia-northeast1_staging-tokyo"
    namespaces: ["frontend", "backend", "database"]
    
  - name: "Development Singapore"
    context: "gke_shopasia-dev_asia-southeast1_dev-singapore" 
    namespaces: ["frontend", "backend", "database"]
```

### Step 3: 자동화된 알람 시스템
```bash
#!/bin/bash
# k9s-health-monitor.sh

# 각 클러스터에서 문제 Pod 탐지
for context in prod-seoul staging-tokyo dev-singapore; do
    kubectl config use-context $context
    
    # 문제 Pod 찾기
    problem_pods=$(kubectl get pods --all-namespaces \
        --field-selector status.phase!=Running \
        --no-headers | wc -l)
    
    if [ $problem_pods -gt 0 ]; then
        echo "🚨 $context: $problem_pods 개의 문제 Pod 발견"
        
        # k9s로 자동 분석 실행
        k9s --context $context --command "pods --all-namespaces"
    fi
done
```

---

## 📝 Week 2 실습 과제

### 과제 1: 멀티 클러스터 환경 구축 (40점)

#### 목표
3개의 독립적인 쿠버네티스 클러스터를 구축하고 k9s로 통합 관리

#### 요구사항
1. **클러스터 3개 생성**
   ```bash
   # 프로덕션 시뮬레이션 (minikube 또는 kind)
   minikube start -p prod --memory=4096 --cpus=2
   minikube start -p staging --memory=2048 --cpus=1  
   minikube start -p dev --memory=1024 --cpus=1
   ```

2. **각 클러스터별 네임스페이스 구성**
   - production: frontend, backend, database, monitoring
   - staging: frontend, backend, testing
   - development: dev, experimental

3. **클러스터별 샘플 워크로드 배포**
   - nginx (frontend)
   - redis (database/backend)
   - busybox (testing/debugging)

#### 제출물
- 각 클러스터의 k9s 스크린샷 (클러스터당 3장)
- kubectl config 설정 파일
- 클러스터 간 전환 과정을 담은 비디오 (5분 이내)

### 과제 2: Control Plane 건강도 체크 시스템 (35점)

#### 목표
k9s를 활용해 Control Plane 컴포넌트의 건강도를 체크하는 체크리스트 작성

#### 체크리스트 항목
1. **API Server 상태**
   - Pod 상태 및 재시작 횟수
   - CPU/메모리 사용률
   - 최근 로그에서 에러 메시지 유무

2. **etcd 클러스터 상태**
   - etcd Pod 상태
   - 디스크 사용률 (describe에서 확인)
   - 클러스터 멤버십 상태

3. **Scheduler 효율성**
   - 스케줄링 지연 시간
   - 스케줄링 실패 횟수
   - 노드 배치 패턴 분석

4. **Controller Manager 동작**
   - 컨트롤러별 처리 성능
   - 리소스 동기화 상태
   - 에러/경고 이벤트 발생 빈도

#### 제출물
- 체크리스트 체크 결과서 (Excel 또는 Markdown)
- 각 컴포넌트별 k9s 스크린샷 증빙
- 발견된 문제점과 해결 방안

### 과제 3: 장애 시뮬레이션과 복구 (25점)

#### 시나리오
의도적으로 다양한 장애 상황을 만들고 k9s로 진단 및 복구

#### 장애 시나리오
1. **Pod CrashLoopBackOff 만들기**
   ```bash
   kubectl run crash-test --image=busybox --restart=Always -- /bin/sh -c "exit 1"
   ```

2. **이미지 Pull 실패 만들기**
   ```bash
   kubectl run image-fail --image=nonexistent:latest
   ```

3. **리소스 부족 만들기**
   ```bash
   kubectl run resource-test --image=nginx --requests='memory=10Gi'
   ```

4. **네트워크 문제 시뮬레이션**
   ```bash
   # 잘못된 Service selector 설정
   kubectl create service clusterip broken-svc --tcp=80:80
   ```

#### 요구사항
- 각 장애에 대한 k9s 진단 과정 기록
- 문제 원인 식별 방법 설명
- 해결 과정과 검증 방법 기록

---

## 🧪 Self-Assessment 퀴즈

### 퀴즈 1: 아키텍처 이해 (25점)

**Q1.1**: API Server가 응답하지 않을 때 k9s에서 가장 먼저 확인해야 할 것은?
- A) etcd Pod의 상태
- B) 네트워크 연결성
- C) kube-apiserver Pod의 상태와 로그
- D) Scheduler 동작 여부

**정답: C**
**해설**: API Server는 모든 요청의 진입점이므로, kube-apiserver Pod 자체의 상태를 먼저 확인해야 합니다. `:pods -n kube-system` → `/apiserver` → `d`와 `l`로 상태와 로그를 확인하는 것이 첫 번째 단계입니다.

---

**Q1.2**: etcd 장애로 인한 클러스터 문제를 k9s에서 식별하는 방법은?
- A) 모든 Pod가 Pending 상태가 되고, API Server가 읽기 전용 모드로 전환
- B) 새로운 리소스 생성이 불가능하고, 기존 상태 변경이 저장되지 않음
- C) Scheduler가 동작하지 않고, Controller Manager가 재시작을 반복
- D) 위의 모든 증상이 복합적으로 나타남

**정답: D**
**해설**: etcd는 클러스터의 모든 상태를 저장하는 핵심 컴포넌트입니다. etcd 장애 시에는 새 리소스 생성 불가, 상태 변경 불가, 컴포넌트 간 통신 장애 등 모든 증상이 복합적으로 나타납니다.

---

### 퀴즈 2: 고급 네비게이션 (25점)

**Q2.1**: 여러 클러스터 환경에서 k9s 사용 시 가장 안전한 방법은?
- A) 하나의 k9s 인스턴스에서 :ctx 명령으로 클러스터 전환
- B) 클러스터별로 별도의 터미널에서 k9s --context 옵션 사용  
- C) kubectl config use-context로 전환 후 k9s 재시작
- D) 환경변수 KUBECONFIG를 변경하여 사용

**정답: B**
**해설**: 프로덕션 환경에서는 실수를 방지하기 위해 클러스터별로 별도 터미널에서 `k9s --context` 옵션을 사용하는 것이 가장 안전합니다. 이렇게 하면 어느 클러스터에서 작업 중인지 명확히 인지할 수 있습니다.

---

**Q2.2**: k9s에서 특정 패턴의 리소스를 효율적으로 찾는 방법은?
- A) 해당 리소스 화면에서 `/패턴` 사용
- B) `shift+:` 명령 팔레트 활용
- C) 정규식을 지원하는 검색 활용  
- D) 모든 방법이 유효하며 상황에 따라 선택

**정답: D**
**해설**: k9s는 다양한 검색 방법을 제공합니다. 간단한 검색은 `/`, 리소스 타입이 불확실할 때는 `shift+:`, 복잡한 패턴 매칭은 정규식을 활용하면 됩니다.

---

### 퀴즈 3: 실전 시나리오 (30점)

**Q3.1**: 다음 상황에서 k9s를 사용한 가장 효율적인 장애 대응 순서는?

**상황**: 프로덕션 환경에서 특정 마이크로서비스 API 응답 시간이 급증하고 있습니다.

- A) `:pods` → 문제 서비스 확인 → `:svc` → 서비스 엔드포인트 확인 → `:hpa` → 스케일링 상태 확인
- B) `:no` → 노드 리소스 확인 → `:pods` → CPU 사용률 정렬 → 문제 Pod 로그 확인  
- C) `:events` → 최근 경고 확인 → 문제 영역 식별 → 해당 리소스 상세 분석
- D) `:deploy` → 문제 서비스 확인 → `:pods` → 로그 분석 → `:svc` → 엔드포인트 확인

**정답: C**
**해설**: 전체적인 시스템 장애의 경우 `:events`로 최근 발생한 경고나 에러를 먼저 확인하여 문제 영역을 식별한 후, 해당 영역을 집중 분석하는 것이 가장 효율적입니다.

---

**Q3.2**: k9s에서 멀티 클러스터 환경의 배포 상태를 비교할 때 가장 체계적인 방법은?
- A) 각 클러스터에서 `:deploy`로 확인 후 수동으로 비교
- B) kubectl 명령어로 배포 정보 추출 후 diff 도구 사용
- C) k9s의 context 전환 기능으로 빠른 비교 후 표준화된 체크리스트 작성
- D) 모니터링 도구 연동으로 자동화된 비교 대시보드 구축

**정답: C**
**해설**: 실시간 운영 상황에서는 k9s의 빠른 context 전환 기능을 활용하되, 체계적인 비교를 위해 표준화된 체크리스트를 사용하는 것이 실용적이고 효율적입니다.

---

### 퀴즈 4: 고급 기능 활용 (20점)

**Q4.1**: k9s에서 HPA(Horizontal Pod Autoscaler)가 예상대로 동작하지 않을 때 확인 순서는?
- A) `:hpa` → `:pods` → `:metrics` → 메트릭 서버 상태 확인
- B) `:hpa` → describe → events 확인 → metrics-server Pod 상태 확인 → 대상 Pod 리소스 확인
- C) `:deploy` → HPA 대상 확인 → `:hpa` → 스케일링 이벤트 확인
- D) `:no` → 노드 용량 확인 → `:hpa` → 임계값 설정 확인

**정답: B**
**해설**: HPA 문제 해결은 HPA 오브젝트의 describe에서 이벤트와 조건을 먼저 확인하고, metrics-server의 동작 상태, 그리고 대상 Pod의 실제 리소스 사용률을 순차적으로 확인해야 합니다.

---

## 🎯 다음 주 예고

Week 3에서는 **"k9s Pod & Container 생명주기 완벽 관리"**를 다룹니다:

- Pod 생명주기의 모든 단계 심층 분석
- Init Container, Sidecar 패턴의 실전 활용
- k9s를 통한 컨테이너 레벨 디버깅 기법
- 리소스 제한과 Quality of Service 최적화
- 실전: Netflix 스케일 마이크로서비스 Pod 관리

### 사전 준비사항
1. Week 2 과제 완료 및 멀티 클러스터 환경 유지
2. Docker 기본 개념 복습 (컨테이너 라이프사이클)
3. 쿠버네티스 Pod 스펙 문서 읽기
4. 리눅스 프로세스와 네임스페이스 개념 숙지

---

**"클러스터를 이해하는 자가 k9s를 다스린다."**

Week 2에서 여러분은 쿠버네티스의 내부 구조와 k9s의 고급 기능을 마스터했습니다. 다음 주에는 이 지식을 바탕으로 실제 컨테이너 워크로드를 완벽하게 관리하는 방법을 배우게 됩니다. 준비되셨나요? 🚀