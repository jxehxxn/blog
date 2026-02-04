---
layout: post
title: "IAM Mastery with Authentik: Week 8 - Application Integration (Proxy vs Forward Auth)"
---

사용자도 있고(LDAP), 인증 로직도 있습니다(Flow). 이제 진짜 서비스에 로그인을 붙여야 합니다.
Week 3, 4에서 배운 OIDC, SAML을 지원하는 "착한 앱"들은 그냥 설정만 하면 됩니다.

하지만 세상엔 인증 기능이 아예 없는 "나쁜 앱"들도 많습니다.
*   사내 개발용 대시보드
*   Prometheus, Alertmanager
*   오래된 PHP 웹사이트

이런 앱들을 위해 Authentik은 **Proxy Provider**와 **Forward Auth**라는 강력한 무기를 제공합니다.

---

## 1. The Strategy: How to protect apps?

앱의 인증 지원 여부에 따라 전략이 갈립니다.

1.  **Level 1 (Modern):** OIDC/SAML 지원함. -> **Provider** 생성해서 연동.
2.  **Level 2 (Legacy):** 인증 기능 없음. -> **Proxy Provider**로 감싸버림.
3.  **Level 3 (Complex):** Nginx/Traefik 뒤에 숨어있음. -> **Forward Auth**로 문지기 세움.

---

## 2. Proxy Provider: The Wrapper

앱 앞에 Authentik Outpost(프록시 서버)를 둡니다.
사용자는 앱에 직접 접근 못 하고, 무조건 Authentik Outpost를 거쳐야 합니다.

*   **동작:** 사용자가 접근 -> Outpost가 가로챔 -> 로그인 안 했으면 로그인 페이지로 -> 로그인 성공하면 앱으로 패스.
*   **헤더 주입:** Outpost는 백엔드 앱에게 `X-Authentik-Username: alice` 같은 헤더를 넣어줄 수 있습니다. 앱은 이 헤더만 믿고 로그인 처리를 하면 됩니다.

---

## 3. Forward Auth: The Gatekeeper

이미 Nginx나 Traefik 같은 리버스 프록시를 쓰고 있다면, 굳이 Authentik Outpost를 또 띄워서 경로를 바꿀 필요가 없습니다.
Nginx가 "인증 검사만" Authentik에게 맡기는 방식입니다.

### Flow
1.  **User -> Nginx:** `GET /dashboard`
2.  **Nginx -> Authentik:** "얘 들여보내도 돼?" (내부 호출)
3.  **Authentik -> Nginx:**
    *   (로그인 안 됨) "아니, 401 Unauthorized." -> Nginx가 로그인 창으로 리다이렉트.
    *   (로그인 됨) "응, 200 OK. 그리고 이 헤더(User Info) 같이 보내."
4.  **Nginx -> App:** Authentik이 준 헤더를 붙여서 요청 전달.

이 방식은 기존 인프라를 거의 건드리지 않고 보안을 입힐 수 있어 **DevOps 엔지니어들이 가장 선호**합니다.

---

## 🛠️ Lab: Protecting "Whoami"

아무런 인증 기능이 없는 `whoami` 컨테이너를 띄우고, Authentik Proxy로 보호해 봅니다.

### 1. Target App 실행
```bash
docker run -d --name whoami -p 8080:80 containous/whoami
```
지금은 `http://localhost:8080` 들어가면 그냥 열립니다.

### 2. Provider 생성 (Proxy)
*   Applications -> Providers -> Create: **Proxy Provider**.
*   **Forward Auth (Single Application)** 모드 선택.
*   **External Host:** `http://whoami.localhost` (사용자가 접속할 주소).
*   **Internal Host:** `http://host.docker.internal:8080` (실제 앱 주소).

### 3. Application 생성
*   Applications -> Applications -> Create.
*   Provider에 위에서 만든 Proxy Provider 연결.

### 4. Outpost 연결
*   Applications -> Outposts -> authentik Embedded Outpost 편집.
*   위 Application을 선택해서 적용.

### 5. 테스트
*   `http://whoami.localhost` 접속 시도.
*   Authentik 로그인 화면이 뜨나요?
*   로그인 후 `whoami` 화면이 나오고, 헤더에 `X-Authentik-Username`이 찍혀 있나요?

---

## 📝 8주차 과제: Forward Auth with Nginx

(Nginx 지식이 필요합니다)

**목표:** 로컬에 Nginx를 띄우고, `auth_request` 모듈을 사용하여 Authentik과 Forward Auth를 구성하세요.

1.  Authentik에서 **Proxy Provider (Forward Auth Domain Level)**를 생성합니다.
2.  Nginx 설정 파일(`nginx.conf`)에 `/outpost.goauthentik.io/` 경로를 설정합니다.
3.  보호하고 싶은 `location /` 블록에 `auth_request /outpost.goauthentik.io/auth/traefik` 설정을 추가합니다.
4.  로그인 안 한 상태에서 접속 시 Authentik으로 리다이렉트 되는지 확인하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 인증 기능이 없는 레거시 웹 애플리케이션에 SSO를 적용하기 위해, HTTP 헤더로 사용자 정보를 전달하는 방식을 무엇이라 하나요? (Header-based Authentication / Forward Auth)
2.  **Q2.** Authentik의 Proxy Provider가 백엔드 애플리케이션에게 전달하는 기본 헤더 중, 사용자의 아이디를 담고 있는 헤더의 이름은? (`X-Authentik-Username` - 설정에서 변경 가능)
3.  **Q3.** Forward Auth 구성에서 Nginx(리버스 프록시)의 역할은 무엇인가요? (트래픽을 받아 Authentik에게 인증 여부를 묻고, 결과에 따라 차단하거나 백엔드로 전달하는 문지기 역할)

---

이제 모든 앱을 Authentik 뒤에 숨겼습니다.
다음 주, 이 철옹성에 **MFA(Multi-Factor Authentication)**라는 이중 잠금장치를 달아봅시다. 비밀번호가 털려도 안전하게 말이죠.

**Wrap it up.**
