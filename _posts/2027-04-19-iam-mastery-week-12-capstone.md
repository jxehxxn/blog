---
layout: post
title: "IAM Mastery with Authentik: Week 12 - Capstone: Production Deployment"
---

축하합니다. 12주의 긴 여정을 마쳤습니다.
LDAP의 트리 구조부터 OIDC 토큰의 바이트 하나하나, 그리고 Python 정책 코딩까지.
이제 여러분은 어디 가서 **"IAM 전문가"**라고 말하셔도 좋습니다.

마지막 과제는 여러분이 배운 모든 것을 쏟아부어 **"절대 죽지 않는 SSO 시스템"**을 구축하는 것입니다.

---

## 🏆 Capstone Project: Enterprise SSO Architecture

### 1. Requirements

가상의 기업 "ACME Corp"의 SSO 시스템을 구축하세요.

**Infrastructure:**
*   **High Availability:** 단일 장애 지점(SPOF)이 없어야 합니다. Authentik Core와 Worker는 최소 2개 이상의 인스턴스로 실행되어야 합니다.
*   **External DB/Cache:** PostgreSQL과 Redis는 컨테이너 내부가 아닌 외부(Managed Service 또는 별도 클러스터)를 사용한다고 가정합니다.
*   **Load Balancer:** Nginx 또는 Traefik을 앞단에 두고 SSL(HTTPS)을 적용해야 합니다.

**Identity Sources:**
*   **LDAP Sync:** 로컬 OpenLDAP 컨테이너를 띄우고, 사용자 100명을 가상으로 생성하여 동기화하세요.

**Applications:**
*   **App A (Proxy):** `whoami` 컨테이너를 Proxy Provider로 보호.
*   **App B (OIDC):** `Grafana` (혹은 OIDC 지원하는 아무 앱) 연동.

**Security Policies:**
*   **MFA Enforcement:** 모든 사용자는 최초 로그인 시 MFA를 설정해야 함.
*   **Geo-Fencing:** 한국(KR) 이외의 IP 접속 차단 (또는 특정 사설 IP 대역만 허용).

### 2. Deliverables (제출물)

1.  **Architecture Diagram:** Draw.io 등으로 구성도 작성. (LB, Core, Worker, Outpost, DB, Redis 연결 관계 표시)
2.  **Docker Compose File:** HA 구성을 모사한 `docker-compose.yml`. (replicas 설정 등)
3.  **Demo Video:**
    *   LDAP 사용자로 로그인 -> MFA 설정 -> App A 접속 -> App B 접속(SSO됨) 시나리오 시연.
    *   Authentik Core 컨테이너 하나를 강제로 껐을 때(`docker stop`), 여전히 로그인이 잘 되는지 시연.

---

## 3. Production Checklist (마지막 점검)

이걸 체크하지 않고 배포하면 밤새워야 할 수도 있습니다.

*   [ ] **SECRET_KEY 보관:** `.env` 파일이 Git에 올라가진 않았나요? (Vault 등을 써야 합니다)
*   [ ] **HTTPS 필수:** HTTP로 OIDC를 쓰면 토큰이 다 털립니다. Let's Encrypt라도 쓰세요.
*   [ ] **Backup Strategy:** PostgreSQL 덤프는 주기적으로 뜨고 있나요?
*   [ ] **Monitoring:** Authentik 자체의 메트릭(`/metrics`)을 Prometheus로 수집하고 있나요? (CPU, 메모리, 로그인 지연 시간)
*   [ ] **Email 설정:** 비밀번호 찾기 메일이 스팸함으로 가지 않나요? (SPF/DKIM 설정)

---

## 🎓 Closing Remarks

보안은 "불편함"이 아닙니다. 보안은 **"비즈니스를 안전하게 달릴 수 있게 하는 브레이크"**입니다. 브레이크가 좋은 차일수록 더 빨리 달릴 수 있죠.

여러분이 구축한 Authentik 시스템 덕분에, 동료들은 수많은 비밀번호의 고통에서 해방되고, 회사는 중앙화된 통제권을 갖게 되었습니다.

이 지식을 가지고 현업으로 돌아가십시오. 그리고 가장 안전하고 편리한 문을 만드십시오.
수고하셨습니다.

**Secure. Simple. Scalable.**
