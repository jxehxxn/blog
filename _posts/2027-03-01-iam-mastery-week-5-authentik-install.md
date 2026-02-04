---
layout: post
title: "IAM Mastery with Authentik: Week 5 - Authentik Installation & Architecture"
---

프로토콜 공부하느라 고생하셨습니다. 이제 실전입니다.
우리는 단순한 '설치'가 아니라, **'프로덕션 레벨의 아키텍처'**를 이해하고 구축할 것입니다.

Authentik은 단순한 Go 바이너리 하나가 아닙니다. Python(Django)과 Go가 섞여 있고, Redis와 PostgreSQL에 의존하며, 작업자(Worker)가 따로 도는 복잡한 시스템입니다.

---

## 1. Authentik Architecture: Under the Hood

Authentik 컨테이너 하나만 띄우면 되는 줄 알았다면 오산입니다.

### Core Components
1.  **Server (Core):**
    *   Python(Django) 기반의 웹 서버입니다.
    *   Admin 인터페이스, API 엔드포인트, SSO 로그인 페이지를 제공합니다.
    *   대부분의 정적 설정과 정책 관리를 담당합니다.
2.  **Worker:**
    *   백그라운드 작업을 처리합니다. (Celery 기반)
    *   LDAP 동기화, 이메일 발송, 이벤트 로그 정리 등을 수행합니다.
    *   이게 죽으면 로그인은 되는데 LDAP 업데이트가 안 되는 기현상이 발생합니다.
3.  **PostgreSQL:**
    *   사용자 정보, 정책, 설정값 등 모든 영구 데이터를 저장합니다.
4.  **Redis:**
    *   세션 캐시, 태스크 큐(Broker) 역할을 합니다. 속도에 핵심적입니다.
5.  **Outpost (Proxy):**
    *   (선택 사항이지만 중요) Go 언어로 작성된 경량 프록시입니다.
    *   앱 앞에 서서 문지기 역할을 하거나(Proxy Provider), LDAP 서버 흉내를 냅니다(LDAP Provider).
    *   Core와 떨어져서 별도 서버에 배포할 수 있습니다. (DMZ 구간 등)

---

## 2. Installation: Docker Compose

가장 표준적인 배포 방식입니다.

### Prerequisites
*   Docker & Docker Compose
*   `pwgen` (비밀번호 생성용)

### Step 1: `docker-compose.yml` & `.env` 다운로드
공식 리포지토리에서 최신 파일을 받습니다.

```bash
wget https://go.authentik.io/docker-compose.yml
wget https://go.authentik.io/env -O .env
```

### Step 2: Secret Key 생성
`.env` 파일을 열어 `PG_PASS`와 `AUTHENTIK_SECRET_KEY`를 채워야 합니다. 절대 기본값(`changeme`)을 쓰지 마세요.

```bash
echo "PG_PASS=$(pwgen -s 40 1)" >> .env
echo "AUTHENTIK_SECRET_KEY=$(pwgen -s 50 1)" >> .env
```

### Step 3: 이메일 설정 (필수)
Authentik은 초기 비밀번호 찾기나 알림을 위해 이메일 서버가 필수입니다. `.env`에서 SMTP 설정을 꼭 하세요. (Gmail이나 AWS SES 등)

```ini
AUTHENTIK_EMAIL__HOST=smtp.gmail.com
AUTHENTIK_EMAIL__PORT=587
AUTHENTIK_EMAIL__USERNAME=myemail@gmail.com
AUTHENTIK_EMAIL__PASSWORD=app-password
AUTHENTIK_EMAIL__USE_TLS=true
AUTHENTIK_EMAIL__FROM=authentik@example.com
```

### Step 4: Run & Init
```bash
docker-compose up -d
```
최초 실행 시 마이그레이션 등으로 시간이 좀 걸립니다.
브라우저로 `http://localhost:9000/if/flow/initial-setup/`에 접속하여 초기 Admin 계정(`akadmin`)을 생성합니다.

---

## 3. High Availability (HA) Considerations

실무에서는 서버 한 대 죽었다고 전 직원 로그인이 안 되면 재앙입니다.

*   **Database:** PostgreSQL을 AWS RDS나 클러스터로 뺍니다.
*   **Cache:** Redis를 ElastiCache나 Redis Cluster로 뺍니다.
*   **Server/Worker:** Kubernetes(Helm Chart)를 사용하여 여러 Pod로 띄웁니다.
*   **Load Balancer:** 앞에 Nginx나 ALB를 두고 트래픽을 분산합니다.

(이 내용은 12주차 Capstone에서 직접 구성해볼 겁니다.)

---

## 4. The First Login: Admin Interface

로그인 후 `Admin Interface` 버튼을 누르면 보이는 대시보드가 여러분의 통제실입니다.

*   **Directory:** 사용자, 그룹, 토큰 관리.
*   **Customization:** 정책, 스테이지, 플로우 관리. (Authentik의 꽃)
*   **Applications:** 연동된 앱 관리.
*   **System:** 테넌트, 브랜드(로고 변경), 로그 확인.

---

## 🛠️ Lab: Branding

여러분의 회사를 위한 Authentik으로 꾸며봅시다.

1.  Admin Interface -> System -> Tenants -> Default Tenant 선택.
2.  **Domain:** 여러분의 도메인으로 설정.
3.  **Branding:**
    *   Logo URL: 회사 로고 이미지 주소.
    *   Title: "My Company SSO".
    *   Flow Background: 멋진 배경화면.
4.  저장 후 시크릿 모드에서 로그인 페이지가 바뀐 것을 확인하세요.

---

## 📝 5주차 과제: GeoIP Setup

보안을 위해 "중국이나 러시아에서 접속하면 차단" 같은 정책을 나중에 만들 겁니다. 이를 위해 Authentik이 접속자의 IP 위치를 알 수 있어야 합니다.

1.  MaxMind GeoIP 계정을 생성하고 라이선스 키를 받으세요. (무료)
2.  Authentik `.env` 파일에 `AUTHENTIK_GEOIP_LICENSE_KEY`를 추가하세요.
3.  컨테이너를 재시작(`docker-compose restart`)하고, Admin 대시보드에서 지도(User Logins)에 위치가 뜨는지 확인하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Authentik의 아키텍처 중, 실제 웹 요청을 처리하지 않고 백그라운드 작업(LDAP 동기화 등)만 수행하는 컴포넌트는? (Worker)
2.  **Q2.** Authentik Core와 떨어져서 별도의 네트워크(예: DMZ)에 배포되어 앱 앞단의 프록시 역할을 할 수 있는 컴포넌트는? (Outpost)
3.  **Q3.** `.env` 파일의 `AUTHENTIK_SECRET_KEY`를 잃어버리거나 변경하면 발생하는 문제는? (기존에 서명된 세션 쿠키, 토큰 등이 모두 무효화되어 모든 사용자가 로그아웃되거나 에러 발생)

---

설치는 끝났습니다. 이제 Authentik의 심장, **"Flow & Stage"** 아키텍처를 이해할 차례입니다. 이게 없으면 Authentik은 껍데기일 뿐입니다. 다음 주에 만납시다.

**Deploy with confidence.**
