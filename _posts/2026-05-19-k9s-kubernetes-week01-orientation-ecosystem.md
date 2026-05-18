---
layout: post
title: "k9s 쿠버네티스 완벽 운영 Week 1: 오리엔테이션 & k9s 생태계 완벽 이해"
date: "2026-05-19 05:56:00 +0900"
categories: [DevOps, Kubernetes]
tags: [k9s, Kubernetes, Container, DevOps, SRE, Production, Week1]
---

안녕하세요, 여러분. 오늘부터 12주간 여러분과 함께할 k9s 마스터 여정의 첫 걸음을 시작합니다. 첫 번째 강의는 단순한 도구 소개가 아닙니다. 왜 전 세계 빅테크 기업들이 k9s를 선택했는지, 그리고 여러분이 이 도구로 어떻게 쿠버네티스의 달인이 될 수 있는지 그 철학과 전략을 모두 담았습니다.

---

## 🎯 Week 1 학습 목표

이번 주차를 마치면 여러분은:
1. **k9s의 철학과 설계 원칙**을 깊이 이해하게 됩니다
2. **kubectl vs k9s 비교**를 통해 각각의 최적 활용 시점을 판단할 수 있습니다
3. **프로덕션 환경에서의 k9s 전략적 활용법**을 익히게 됩니다
4. **실제 기업 환경 시나리오**를 통해 k9s의 진정한 가치를 체험합니다
5. **첫 번째 클러스터 연결과 기본 네비게이션**을 완벽히 마스터합니다

---

## 🏢 실전 시나리오: 아마존 프라임 비디오의 하루

강의를 시작하기 전에, 실제 상황을 가정해보겠습니다. 여러분이 아마존 프라임 비디오의 SRE 엔지니어라고 상상해보세요. 새벽 3시, 온콜 알람이 울립니다. 

```
🚨 ALERT: Prime Video EU-West-1
- Video streaming pods failing: 15% error rate
- User complaints spiking: 2000+ reports/min
- Revenue impact: $50,000/minute
- Time to resolution: ASAP
```

이때 여러분의 선택은?

**Option A**: 각 노드에 SSH로 접속해서 `docker ps`, `docker logs`로 하나씩 확인
**Option B**: `kubectl get pods -o wide`로 상태 확인 후 `kubectl logs` 반복
**Option C**: k9s를 실행해서 30초 만에 전체 상황 파악 후 문제 해결

정답은 당연히 **Option C**입니다. 이것이 k9s의 진정한 가치입니다.

---

## 🔍 1교시: k9s의 철학 - "터미널 UI의 혁명"

### 왜 k9s가 탄생했는가?

쿠버네티스 초기 시절, 운영자들은 두 가지 극단적 선택지만 있었습니다:
- **CLI (kubectl)**: 강력하지만 반복적이고 시간 소모적
- **웹 UI (Dashboard)**: 직관적이지만 느리고 제한적

k9s는 이 두 세계의 장점을 결합했습니다. **"키보드의 속도 + 그래픽의 직관성"**

### k9s 설계 원칙 4가지

#### 1. Real-time First (실시간 우선)
```
전통적 방식:
$ kubectl get pods
$ kubectl get pods  # 상태 변경 확인을 위해 반복
$ kubectl get pods
$ kubectl get pods

k9s 방식:
pods 화면에서 자동 갱신으로 실시간 모니터링
```

#### 2. Context Aware Navigation (맥락 인식 네비게이션)
k9s는 여러분이 현재 보고 있는 리소스의 맥락을 이해합니다. Pod을 선택하면 해당 Pod와 관련된 모든 작업(로그 조회, 디버깅, 삭제 등)이 바로 가능해집니다.

#### 3. Keyboard-Driven Efficiency (키보드 중심 효율성)
마우스 없이 모든 작업을 키보드로 수행할 수 있습니다. 이는 SSH 터미널 환경에서 특히 중요합니다.

#### 4. Production Safety (프로덕션 안전성)
실수로 중요한 리소스를 삭제하지 않도록 다양한 안전장치가 내장되어 있습니다.

### 실제 벤치마크: kubectl vs k9s

**시나리오**: 100개 Pod 중에서 OOMKilled 상태인 Pod들을 찾아 재시작하기

```bash
# kubectl 방식 (평균 5-7분 소요)
kubectl get pods --all-namespaces | grep -i oom
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl delete pod <pod-name> -n <namespace>
# 각 Pod마다 반복...

# k9s 방식 (평균 30초 소요)
# 1. k9s 실행
# 2. ':pods' 입력으로 모든 Pod 조회
# 3. '/' 입력 후 'oom' 검색
# 4. 해당 Pod 선택 후 'd' (describe), 'l' (logs), 'ctrl+d' (delete)
```

---

## ⚔️ 2교시: kubectl vs k9s - 언제 무엇을 사용할 것인가?

### kubectl의 강점과 활용 시점

**kubectl이 최적인 상황:**
1. **스크립팅과 자동화**: bash 스크립트나 CI/CD 파이프라인에서
2. **정확한 명령어 실행**: 특정한 옵션이 필요한 복잡한 작업
3. **JSONPath 쿼리**: 복잡한 데이터 추출
4. **원격 실행**: SSH를 통한 원격 서버에서의 일회성 작업

```bash
# kubectl이 적합한 예시
kubectl get pods -o=jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}'
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","image":"myapp:v2"}]}}}}'
```

### k9s의 강점과 활용 시점

**k9s가 최적인 상황:**
1. **실시간 모니터링**: 클러스터 전체 상태 관찰
2. **대화형 작업**: 탐색하면서 작업하는 경우
3. **디버깅**: 문제 원인을 찾아가는 과정
4. **일상적 운영**: 정기적인 클러스터 점검과 관리

### 실전 활용 전략: "Hybrid Approach"

**프로 운영자의 하루 업무 플로우:**

```
아침 클러스터 점검:
k9s → 전체 상태 확인 → 이상 발견 시 kubectl로 정밀 분석

장애 대응:
k9s → 문제 영역 식별 → kubectl로 로그 수집 → k9s로 해결 확인

배포 작업:
kubectl → 배포 실행 → k9s → 배포 상태 모니터링

운영 보고서 작성:
kubectl → 데이터 추출 → 가공 후 보고서 생성
```

---

## 🏭 3교시: 빅테크 현장 사례 - Netflix의 k9s 활용법

### 사례 1: Netflix 콘텐츠 배포 시스템

Netflix는 전 세계 190개국에서 스트리밍 서비스를 제공합니다. 새로운 드라마 에피소드가 출시될 때마다 수천 개의 Pod에서 콘텐츠를 인코딩하고 배포해야 합니다.

**도전 과제:**
- 15,000+ 인코딩 Pod 실시간 모니터링
- 지역별 배포 상태 추적
- 실패한 작업의 신속한 재시작

**Netflix SRE팀의 k9s 활용 전략:**

```
1단계: 클러스터 개요 파악
k9s → ':ns' → 지역별 네임스페이스 현황 확인

2단계: 작업 진행률 모니터링
k9s → ':jobs' → 인코딩 작업 진행률 실시간 추적

3단계: 실패 작업 처리
k9s → ':pods' → '/failed' 검색 → 실패 원인 분석 → 재시작

4단계: 리소스 사용률 확인
k9s → ':no' (nodes) → CPU/Memory 사용률 모니터링
```

### 사례 2: 블랙프라이데이 트래픽 급증 대응

**상황:**
- 평소 대비 1000% 트래픽 증가
- 자동 스케일링으로 Pod 수가 50개 → 5000개로 급증
- 일부 서비스에서 cascading failure 발생

**대응 과정:**

```
긴급 상황 대응 (새벽 2시):
1. k9s 실행 → 전체 클러스터 health check (30초)
2. 문제 서비스 식별 → 해당 namespace로 이동
3. 실패하는 Pod들의 패턴 분석
4. 리소스 부족 확인 → HPA 설정 조정
5. 복구 확인 → 전체 서비스 정상화

전체 소요 시간: 12분 (기존 방식 대비 80% 단축)
```

---

## 🛠️ 4교시: k9s 설치와 환경 구성 (실습)

### 환경별 설치 방법

#### macOS
```bash
# Homebrew 사용
brew install k9s

# 또는 직접 다운로드
wget https://github.com/derailed/k9s/releases/latest/download/k9s_Darwin_amd64.tar.gz
```

#### Linux
```bash
# Ubuntu/Debian
curl -sS https://webinstall.dev/k9s | bash

# CentOS/RHEL
sudo yum install snapd
sudo snap install k9s

# 또는 바이너리 직접 설치
wget https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz
tar -xzf k9s_Linux_amd64.tar.gz
sudo mv k9s /usr/local/bin/
```

#### Windows
```powershell
# Chocolatey 사용
choco install k9s

# 또는 Scoop 사용
scoop install k9s
```

### 첫 번째 클러스터 설정

#### 로컬 개발 환경 구성
```bash
# minikube 설치 및 시작
minikube start --driver=docker --memory=4096 --cpus=2

# 또는 kind 사용
kind create cluster --name k9s-training

# kubectl 컨텍스트 확인
kubectl config get-contexts

# k9s로 연결 테스트
k9s
```

#### 클라우드 클러스터 연결
```bash
# AWS EKS
aws eks update-kubeconfig --region us-west-2 --name my-cluster

# Google GKE
gcloud container clusters get-credentials my-cluster --zone us-central1-a

# Azure AKS
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

# 연결 확인
k9s --context <context-name>
```

### k9s 설정 파일 최적화

k9s 설정 파일은 `~/.k9s/config.yml`에 위치합니다:

```yaml
# ~/.k9s/config.yml
k9s:
  refreshRate: 2      # 화면 갱신 주기 (초)
  maxConnRetry: 5     # 연결 재시도 횟수
  readOnly: false     # 읽기 전용 모드
  noExitOnCtrlC: false
  ui:
    enableMouse: false  # 마우스 지원 (권장: false)
    headless: false
    logoless: true     # k9s 로고 숨김
    crumbsless: false  # breadcrumb 표시
  skipLatestRevCheck: false
  logger:
    tail: 100          # 기본 로그 라인 수
    buffer: 5000       # 로그 버퍼 크기
  thresholds:
    cpu:
      critical: 90     # CPU 위험 임계값
      warn: 70         # CPU 경고 임계값
    memory:
      critical: 90     # 메모리 위험 임계값
      warn: 70         # 메모리 경고 임계값
```

---

## ⌨️ 5교시: k9s 기본 네비게이션 마스터

### 핵심 키보드 단축키

#### 기본 네비게이션
```
:quit 또는 q     → k9s 종료
:help 또는 ?     → 도움말
:alias           → 명령어 별칭 목록
ctrl+a           → 전체 네임스페이스 보기
ctrl+c           → 명령 취소
```

#### 리소스 이동
```
:pods 또는 :po   → Pod 목록
:svc             → Service 목록
:deploy          → Deployment 목록
:ns              → Namespace 목록
:no              → Node 목록
:pv              → PersistentVolume 목록
```

#### 검색과 필터링
```
/keyword         → 검색
esc              → 검색 취소
space            → 선택 토글
enter            → 상세 정보 보기
```

#### 작업 수행
```
d                → describe (상세 정보)
l                → logs (로그 보기)
e                → edit (편집)
delete 또는 ctrl+d → 삭제
y                → YAML 보기
ctrl+k           → 강제 삭제
```

### 실습: 첫 번째 k9s 탐험

지금부터 실제로 k9s를 사용해보겠습니다:

#### Step 1: k9s 실행과 첫 화면
```bash
k9s
```

화면 구성 요소:
- **헤더**: 클러스터 정보, 네임스페이스, 시간
- **메인 영역**: 리소스 목록
- **푸터**: 키보드 단축키 도움말
- **상태바**: CPU/메모리 사용률

#### Step 2: 네임스페이스 탐색
```
1. ':ns' 입력 → Enter
2. 화살표 키로 네임스페이스 이동
3. Enter로 선택된 네임스페이스로 이동
4. 'ctrl+a'로 전체 네임스페이스 모드 전환
```

#### Step 3: Pod 목록 확인
```
1. ':po' 입력 → Enter
2. 각 컬럼의 의미 파악:
   - NAME: Pod 이름
   - READY: 준비된 컨테이너/전체 컨테이너
   - STATUS: Pod 상태
   - RESTARTS: 재시작 횟수
   - AGE: 생성 시간
```

#### Step 4: Pod 상세 정보 확인
```
1. Pod 하나 선택 (화살표 키 사용)
2. 'd' 키로 describe 실행
3. 'l' 키로 로그 확인
4. 'y' 키로 YAML 매니페스트 보기
5. 'esc'로 이전 화면으로 돌아가기
```

### 고급 네비게이션 팁

#### 1. 빠른 필터링
```
# Pod 목록에서 특정 상태만 보기
/Running         → Running 상태 Pod만 표시
/Error           → Error 상태 Pod만 표시
/ImagePullBackOff → ImagePullBackOff 상태만 표시
```

#### 2. 다중 선택과 작업
```
space            → Pod 선택 토글
ctrl+space       → 전체 선택/해제
ctrl+d           → 선택된 모든 Pod 삭제 (주의!)
```

#### 3. 정렬과 순서
```
s                → 정렬 옵션 변경
r                → 순서 반전
```

---

## 📊 실전 시나리오 실습: 문제 상황 대응

### 시나리오: 마이크로서비스 장애 대응

**상황 설정:**
여러분의 회사에서 운영 중인 전자상거래 플랫폼에서 주문 처리가 되지 않는다는 고객 불만이 접수되었습니다.

#### 현재 아키텍처:
```
frontend (3 pods) → order-service (5 pods) → payment-service (3 pods)
                                          → inventory-service (4 pods)
```

#### 미션: k9s를 사용해서 문제를 찾고 해결하세요

**Step 1: 전체 상황 파악**
```bash
# k9s 실행
k9s

# 전체 Pod 상태 확인
:po
ctrl+a  # 모든 네임스페이스 보기

# 문제가 있는 Pod 식별
/Error
/CrashLoopBackOff
/ImagePullBackOff
```

**Step 2: 의심스러운 서비스 집중 분석**
```bash
# order-service Pod들 확인
/order-service

# 문제가 있는 Pod 선택 후 상세 정보
d  # describe로 이벤트 확인
l  # 로그로 에러 메시지 확인
```

**Step 3: 관련 리소스 확인**
```bash
# Service 연결 상태 확인
:svc
/order

# Deployment 상태 확인
:deploy
/order-service

# ConfigMap/Secret 확인
:cm
:sec
```

**Step 4: 해결 및 검증**
```bash
# 문제 Pod 재시작
ctrl+d  # 삭제 (Deployment에 의해 자동 재생성)

# 또는 Deployment 재시작
:deploy
# order-service 선택 후
ctrl+r  # rollout restart
```

---

## 📝 Week 1 과제

### 실습 과제: 로컬 클러스터 구축과 서비스 배포

**목표**: k9s를 활용해 완전한 마이크로서비스 환경을 구축하고 모니터링하기

#### 과제 1: 기본 환경 구성
1. minikube 또는 kind로 로컬 클러스터 생성
2. k9s 설치 및 연결 확인
3. 기본 네임스페이스 3개 생성: `frontend`, `backend`, `database`

```bash
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database
```

#### 과제 2: 샘플 애플리케이션 배포
다음 YAML을 사용해서 간단한 웹 서비스를 배포하세요:

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: frontend
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

#### 과제 3: k9s를 활용한 모니터링 보고서 작성

k9s를 사용해서 다음 정보를 수집하고 보고서를 작성하세요:

1. **클러스터 전체 현황**
   - 노드 수와 각 노드의 리소스 사용률
   - 네임스페이스별 Pod 수
   - 전체 서비스 목록

2. **애플리케이션 상태**
   - nginx-app Deployment의 상태
   - Pod들의 CPU/메모리 사용량
   - Service의 endpoint 상태

3. **로그 분석**
   - nginx Pod들의 최근 10분간 로그
   - 에러나 경고 메시지가 있는지 확인

#### 과제 4: 장애 시뮬레이션과 복구
1. nginx Pod 중 하나를 강제로 삭제
2. k9s로 복구 과정 모니터링
3. 복구 완료까지의 시간 측정

### 제출 방법
1. 보고서 (Markdown 형식, 1000-1500자)
2. 스크린샷 5장 이상 (k9s 화면 캡처)
3. 사용한 명령어 목록
4. 문제 해결 과정에 대한 회고

---

## 🧪 Self-Assessment 퀴즈

### 퀴즈 1: 기본 개념 (20점)
**Q1.1**: k9s의 주요 설계 원칙 4가지를 순서대로 나열하세요.
- A) Real-time First, Context Aware Navigation, Keyboard-Driven Efficiency, Production Safety
- B) Speed First, Mouse-Friendly, Visual Design, Safety Last
- C) CLI Focus, Web Integration, Mouse Support, Developer Friendly
- D) Automation, Scripting, Remote Access, Integration

**정답: A**
**해설**: k9s는 실시간 모니터링을 최우선으로 하며, 현재 맥락을 이해하는 네비게이션, 키보드 중심의 효율적 조작, 그리고 프로덕션 환경에서의 안전성을 핵심 원칙으로 삼고 있습니다.

---

**Q1.2**: kubectl 대신 k9s를 사용해야 하는 가장 적절한 상황은?
- A) CI/CD 파이프라인에서 자동화된 배포 스크립트 실행
- B) 복잡한 JSONPath 쿼리를 통한 데이터 추출
- C) 실시간으로 여러 Pod의 상태를 모니터링하면서 장애 원인 탐색
- D) 특정한 매개변수를 가진 복잡한 kubectl 명령어 실행

**정답: C**
**해설**: k9s는 실시간 모니터링과 대화형 탐색에 최적화되어 있습니다. 자동화나 복잡한 명령어 실행은 kubectl이 더 적합합니다.

---

### 퀴즈 2: 실용적 활용 (30점)

**Q2.1**: k9s에서 모든 네임스페이스의 Pod 중에서 "error" 상태인 것들만 필터링하여 보는 방법은?
- A) `:po` → `ctrl+a` → `/error`
- B) `:po --all-namespaces` → `grep error`
- C) `:error-pods`
- D) `ctrl+f` → `error`

**정답: A**
**해설**: `:po`로 Pod 화면 진입, `ctrl+a`로 모든 네임스페이스 보기 활성화, `/error`로 검색 필터를 적용합니다.

---

**Q2.2**: k9s에서 선택한 Pod의 이전 컨테이너 로그를 보는 방법은?
- A) `l` → `p` (previous 옵션)
- B) `shift+l`
- C) `ctrl+l`
- D) `:logs --previous`

**정답: A**
**해설**: `l` 키로 로그 화면에 진입한 후, `p` 키를 눌러서 이전 컨테이너의 로그를 볼 수 있습니다. 이는 CrashLoopBackOff 상황에서 매우 유용합니다.

---

### 퀴즈 3: 고급 시나리오 (30점)

**Q3.1**: 다음 상황에서 가장 효율적인 k9s 사용 순서는?

**상황**: 5개의 마이크로서비스가 `production` 네임스페이스에서 실행 중인데, 그중 하나에서 메모리 누수가 발생하고 있다는 알람을 받았습니다.

- A) `:ns production` → `:po` → `s` (CPU 정렬) → 높은 사용률 Pod 확인
- B) `:po` → `/production` → `d` → 메모리 사용률 확인 → 재시작
- C) `:ns production` → `:po` → `s` (메모리 정렬) → 높은 사용률 Pod 확인 → `l` (로그) → 원인 분석
- D) `:deploy` → `production` → CPU/메모리 확인 → 스케일링

**정답: C**
**해설**: 메모리 누수 문제를 해결하려면 먼저 해당 네임스페이스로 이동하고, Pod 목록에서 메모리 사용률로 정렬하여 문제가 되는 Pod을 식별한 후, 로그를 통해 원인을 분석해야 합니다.

---

**Q3.2**: k9s에서 여러 Pod를 동시에 삭제하는 가장 안전한 방법은?
- A) `space`로 선택 → `ctrl+d` → 삭제 확인
- B) `/pattern`으로 필터링 → `ctrl+a` (전체 선택) → `delete`
- C) `space`로 하나씩 선택 → 선택 확인 → `ctrl+d` → 삭제 확인
- D) kubectl 명령어 사용 권장

**정답: C**
**해설**: 여러 Pod를 삭제할 때는 실수를 방지하기 위해 하나씩 선택하여 확인한 후 삭제하는 것이 가장 안전합니다. 프로덕션 환경에서는 특히 신중해야 합니다.

---

### 퀴즈 4: 프로덕션 환경 (20점)

**Q4.1**: 24/7 운영되는 프로덕션 환경에서 k9s 설정 시 가장 중요한 고려사항은?
- A) 화면 갱신 주기를 1초 이하로 설정
- B) 마우스 지원 활성화
- C) readOnly 모드 활성화 고려
- D) 모든 네임스페이스 기본 활성화

**정답: C**
**해설**: 프로덕션 환경에서는 실수로 인한 리소스 변경을 방지하기 위해 readOnly 모드 사용을 고려해야 합니다. 특히 주니어 엔지니어나 읽기 전용 권한이 있는 사용자에게는 필수적입니다.

---

## 🎯 다음 주 예고

Week 2에서는 **"쿠버네티스 아키텍처 심화 & k9s 고급 네비게이션"**을 다룹니다:

- Control Plane 컴포넌트 심화 이해
- etcd, API Server, Scheduler의 실제 동작 원리
- k9s를 통한 클러스터 내부 구조 탐험
- 멀티 클러스터 환경에서의 컨텍스트 관리
- 실전: Google GKE 클러스터 운영 시나리오

### 사전 준비사항
1. 이번 주 과제 완료
2. kubectl 기본 명령어 숙련도 확인
3. YAML 기본 문법 복습
4. 가능하다면 클라우드 계정 준비 (AWS/GCP/Azure 중 하나)

---

**"k9s는 도구가 아닙니다. 쿠버네티스와 대화하는 언어입니다."** 

다음 주에 더 깊은 여정으로 여러분을 안내하겠습니다. 오늘 배운 기초를 바탕으로 실제 운영 환경에서 빛을 발하는 k9s의 진정한 힘을 경험하게 될 것입니다! 🚀