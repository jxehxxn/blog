---
layout: post
title: "Kubernetes Package Management with Helm: Week 2 - Helm Fundamentals & Architecture"
---

Week 1에서 Helm의 필요성을 느꼈습니다. 이제 Helm의 해부학 시간입니다.
Helm은 겉으로 보기엔 단순한 CLI 도구 같지만, 내부적으로는 체계적인 구조를 가지고 있습니다.

특히 Helm v2에서 v3로 넘어오면서 가장 큰 변화인 **"Tiller의 제거"**를 이해하는 것이 중요합니다. (보안 면접 단골 질문!)

---

## 1. Helm Architecture: v2 vs v3

### The Tiller Problem (v2)
옛날 Helm v2는 클러스터 내부에 `Tiller`라는 서버 모듈(Pod)을 설치해야 했습니다.
*   **Helm CLI -> Tiller -> Kubernetes API**
*   **문제점:** Tiller는 클러스터 전체에 대한 슈퍼 관리자 권한(ClusterAdmin)을 가지고 있었습니다. 해커가 Tiller만 장악하면 클러스터 전체가 털립니다. 보안상 최악이었죠.

### Client-Only Architecture (v3)
Helm v3는 Tiller를 없앴습니다.
*   **Helm CLI -> Kubernetes API**
*   **장점:** `kubectl`을 쓰는 사용자의 권한(kubeconfig)을 그대로 사용합니다. 사용자가 권한이 없으면 Helm도 배포 못 합니다. (RBAC 준수)
*   **상태 저장:** 배포된 릴리스 정보(State)를 Tiller 메모리가 아닌, 쿠버네티스 **Secret** 객체로 네임스페이스마다 따로 저장합니다.

---

## 2. Core Concepts: Chart, Config, Release

Helm을 지탱하는 3대 요소입니다.

1.  **Chart (차트):** 붕어빵 틀.
    *   앱 설치에 필요한 템플릿 파일들의 묶음. (`/templates`)
    *   기본 설정값. (`values.yaml`)
2.  **Config (설정):** 팥, 슈크림, 피자 토핑.
    *   사용자가 덮어씌운 설정값. (`--set` 또는 `my-values.yaml`)
3.  **Release (릴리스):** 팥 붕어빵, 슈크림 붕어빵.
    *   Chart + Config가 합쳐져서 클러스터에 실제로 실행된 인스턴스.
    *   같은 `nginx` 차트로 `frontend` 릴리스와 `backend` 릴리스를 따로 만들 수 있습니다.

---

## 3. Directory Structure of a Chart

`helm create my-chart`를 치면 다음과 같은 폴더가 생깁니다.

```
my-chart/
├── Chart.yaml          # 차트의 메타데이터 (이름, 버전)
├── values.yaml         # 기본 설정값 (Default Values)
├── charts/             # 의존성 차트 (Subcharts)
└── templates/          # 실제 리소스 템플릿들
    ├── deployment.yaml
    ├── service.yaml
    ├── _helpers.tpl    # 재사용 가능한 템플릿 조각 (함수)
    └── NOTES.txt       # 설치 후 사용자에게 보여줄 메시지
```

이 구조는 표준입니다. 마음대로 바꾸면 안 됩니다.

---

## 🛠️ Lab: Helm CLI Mastery

Helm CLI의 핵심 명령어를 손에 익혀봅시다.

### 1. Repo 관리
```bash
helm repo add stable https://charts.helm.sh/stable
helm repo list
helm search repo mysql
```

### 2. Install & Upgrade
```bash
# 설치 (릴리스 이름: my-release)
helm install my-release bitnami/mysql

# 설정 변경해서 업그레이드
helm upgrade my-release bitnami/mysql --set auth.rootPassword=secretpassword
```

### 3. History & Rollback
가장 강력한 기능입니다. 배포하다가 에러 나면 바로 되돌릴 수 있습니다.
```bash
# 배포 이력 확인
helm history my-release

# 1번 버전(Revision)으로 롤백
helm rollback my-release 1
```

### 4. Uninstall
```bash
# 삭제 (하지만 --keep-history 옵션을 주면 기록은 남길 수 있음)
helm uninstall my-release
```

---

## 📝 2주차 과제: Analyze Existing Chart

**목표:** 상용 차트를 분석하여 구조를 파악하세요.

1.  `bitnami/nginx` 차트를 다운로드(`helm pull bitnami/nginx --untar`)합니다.
2.  압축 풀린 폴더에 들어가서 `values.yaml`을 열어보세요.
3.  이미지 태그(`image.tag`), 서비스 타입(`service.type`), 리소스 제한(`resources`) 설정이 기본적으로 어떻게 되어있는지 찾아서 적어내세요.
4.  `templates/deployment.yaml`을 열어서, 위 `values.yaml`의 값들이 어디에 매핑(`{{ .Values... }}`)되어 있는지 한 군데만 찾아서 캡처하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Helm v3에서 릴리스 정보(Release State)는 쿠버네티스의 어떤 리소스 타입으로 저장되나요? (Secret)
2.  **Q2.** 차트 버전(Chart Version)과 앱 버전(App Version)의 차이는 무엇인가요? (차트 버전은 패키지 자체의 버전, 앱 버전은 그 안에 들어가는 Nginx/MySQL 등의 실제 소프트웨어 버전)
3.  **Q3.** `helm upgrade` 시 `--atomic` 플래그의 역할은 무엇인가요? (배포 실패 시 자동으로 롤백해주는 옵션)

---

다음 주, 남이 만든 차트 말고 **"나만의 차트"**를 바닥부터 만들어봅니다.
Go Template 언어의 세계로 오신 것을 환영합니다.

**Helm is the way.**
