---
layout: post
title: "Kubernetes Package Management with Helm: Week 5 - Dependency Management & Subcharts"
---

웹 애플리케이션 차트를 만들었습니다. 그런데 이 앱은 **MySQL**이 없으면 안 돌아갑니다.
사용자가 내 차트를 설치할 때, MySQL도 같이 설치되게 하려면 어떻게 해야 할까요?

1.  사용자한테 "MySQL 먼저 설치하고 오세요"라고 한다. (불친절)
2.  내 차트 안에 MySQL Deployment YAML을 복사해서 넣는다. (유지보수 지옥)
3.  **Helm Dependency** 기능을 쓴다. (정답)

오늘은 남이 잘 만들어둔 차트(Bitnami MySQL 등)를 내 차트의 **부품(Subchart)**으로 가져다 쓰는 법을 배웁니다.

---

## 1. `Chart.yaml` Dependencies

`Chart.yaml` 파일에 `dependencies` 섹션을 추가하면 됩니다.

```yaml
dependencies:
  - name: mysql
    version: 8.8.2
    repository: https://charts.bitnami.com/bitnami
    condition: mysql.enabled  # 이 값이 true일 때만 설치됨
    tags:
      - database
```

### Dependency Update
설정만 한다고 끝이 아닙니다. 실제 차트 파일(.tgz)을 다운로드받아야 합니다.
```bash
helm dependency update ./my-chart
```
이러면 `charts/` 폴더 안에 `mysql-8.8.2.tgz` 파일이 생깁니다.

---

## 2. Configuration: Overriding Subchart Values

서브차트(MySQL)의 `values.yaml`을 내 입맛대로 바꾸고 싶습니다. (예: DB 비밀번호 설정)
내 차트(Parent)의 `values.yaml`에서 **서브차트 이름**으로 섹션을 만들면 됩니다.

```yaml
# my-chart/values.yaml

# 서브차트 이름(mysql) 아래에 설정
mysql:
  enabled: true
  auth:
    rootPassword: "secretpassword"
    database: "mydb"
```

Helm은 이 값들을 MySQL 차트의 `values.yaml`에 덮어씌워서 렌더링합니다.

---

## 3. Global Values

여러 서브차트가 공통으로 써야 하는 값이 있습니다. (예: Docker Registry 주소, 환경 이름)
`global` 키워드를 사용합니다.

```yaml
# my-chart/values.yaml
global:
  imageRegistry: "docker.io"
  env: "prod"

mysql:
  image:
    registry: "docker.io" # 이렇게 따로 안 써도 됨
```

서브차트 템플릿에서는 `.Values.global.env` 로 접근할 수 있습니다.

---

## 4. Subcharts (직접 만들기)

외부 차트뿐만 아니라, 내가 만든 작은 차트를 폴더째로 `charts/` 안에 넣을 수도 있습니다.
이것을 **Subchart**라고 합니다.

*   `my-chart/charts/backend/`
*   `my-chart/charts/frontend/`

이렇게 하면 거대한 MSA 시스템 전체를 하나의 `Umbrella Chart`(우산 차트)로 묶어서 배포할 수 있습니다. `helm install my-system` 한 방에 프론트, 백엔드, DB가 다 뜹니다.

---

## 🛠️ Lab: WordPress with MySQL

WordPress는 DB가 필수입니다. 의존성을 실습하기 가장 좋은 예제입니다.

1.  **차트 생성:** `helm create my-wordpress`
2.  **의존성 추가:** `Chart.yaml`에 `bitnami/mariadb`를 추가합니다.
3.  **다운로드:** `helm dep up`
4.  **설정 오버라이드:** `values.yaml`에 `mariadb.auth.rootPassword` 등을 설정합니다.
5.  **연결:** `templates/deployment.yaml`의 환경 변수(WORDPRESS_DB_HOST)에 MariaDB 서비스 이름을 넣어줍니다. (보통 `{{ .Release.Name }}-mariadb` 형태)

---

## 📝 5주차 과제: The Umbrella Chart

**목표:** Frontend(Nginx)와 Backend(Node.js)를 포함하는 Umbrella Chart를 구성하세요.

1.  `charts/frontend` 폴더에 Nginx 차트를 만듭니다. (Week 3에서 만든 것 활용)
2.  `charts/backend` 폴더에 Node.js 차트를 만듭니다. (간단한 Deployment)
3.  부모 `values.yaml`에서 두 서브차트의 `replicaCount`를 각각 제어할 수 있게 설정하세요.
    ```yaml
    frontend:
      replicaCount: 2
    backend:
      replicaCount: 1
    ```
4.  `helm install` 시 Pod가 총 3개(Frontend 2개, Backend 1개)가 뜨는지 확인하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** `helm dependency update` 명령을 실행하면 다운로드된 서브차트 파일(.tgz)은 어느 디렉토리에 저장되나요? (`charts/`)
2.  **Q2.** 부모 차트의 `values.yaml`에서 서브차트 `redis`의 설정을 변경하려면 어떤 키(Key) 아래에 값을 적어야 하나요? (`redis:`)
3.  **Q3.** 여러 서브차트 간에 공통으로 공유되는 변수(예: 클러스터 도메인 등)를 정의하기 위해 사용하는 예약어는? (`global`)

---

다음 주, 이제 차트 만드는 법은 마스터했습니다.
하지만 매번 `helm upgrade` 치기 귀찮지 않나요? **Skaffold**를 이용해 저장(Save)하는 순간 자동으로 배포되는 **Local Development** 환경을 구축해봅니다.

**Dependencies resolved.**
