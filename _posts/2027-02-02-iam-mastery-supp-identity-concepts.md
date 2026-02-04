---
layout: post
title: "IAM Mastery with Authentik: Supplement - Identity Concepts (AuthN vs AuthZ)"
---

Week 1에서 **인증(Authentication)**과 **인가(Authorization)**의 차이를 간단히 짚고 넘어갔습니다. 하지만 실무에서는 이 두 가지가 미묘하게 섞여 있어 혼란을 줍니다.

"로그인은 성공했는데 왜 403 Forbidden이 뜨죠?"
"이 토큰만 있으면 관리자 페이지 들어갈 수 있나요?"

이 질문에 명확히 답하기 위해, 보안 엔지니어가 반드시 알아야 할 **Identity Concepts**를 정리합니다.

---

## 1. The Three Pillars of IAM

IAM은 크게 세 가지 기둥으로 이루어집니다.

### 1) Identification (식별)
*   **질문:** "누구라고 주장하는가?"
*   **행위:** ID(Username)를 입력하는 것.
*   **비유:** "저 홍길동인데요." (아직 증명 안 됨)

### 2) Authentication (인증, AuthN)
*   **질문:** "그 주장이 사실인가?"
*   **행위:** 비밀번호, OTP, 지문 등을 제시하여 증명하는 것.
*   **비유:** "여기 제 신분증과 지문입니다." (확인됨)

### 3) Authorization (인가, AuthZ)
*   **질문:** "그래서 무엇을 할 수 있는가?"
*   **행위:** 시스템이 권한 목록(ACL, Policy)을 확인하여 접근을 허용/거부하는 것.
*   **비유:** "홍길동님은 VIP 룸에 들어갈 수 있습니다."

---

## 2. Access Control Models (접근 제어 모델)

인가(AuthZ)를 구현하는 방식은 여러 가지가 있습니다. Authentik은 이 중 **RBAC**과 **ABAC**을 강력하게 지원합니다.

### DAC (Discretionary Access Control)
*   **설명:** 주인이 맘대로 권한을 줌.
*   **예시:** 리눅스 파일 권한 (`chmod 777`). 파일 주인이 읽기/쓰기 권한을 결정함.

### RBAC (Role-Based Access Control) - **표준**
*   **설명:** 사람에게 권한을 주는 게 아니라, **'역할(Role)'**에 권한을 주고 사람을 역할에 할당함.
*   **예시:**
    *   `Developer` 역할: 서버 재시작 권한 O
    *   `Alice`: `Developer` 역할임.
    *   결론: `Alice`는 서버 재시작 가능.
*   **장점:** 관리가 편함. 입/퇴사 시 역할만 바꾸면 됨.

### ABAC (Attribute-Based Access Control) - **Authentik의 강점**
*   **설명:** 속성(Attribute)을 기반으로 동적으로 결정함.
*   **예시:** "부서가 'IT'이고(User Attribute), 현재 시간이 '업무 시간'이고(Environment Attribute), 접속 IP가 '사내망'이면(Context Attribute) 허용."
*   **Authentik:** Python Policy를 이용해 "IP가 한국이면 허용" 같은 ABAC을 구현할 수 있습니다.

---

## 3. Session vs Token

로그인 후 "내가 인증된 상태다"라는 걸 어떻게 유지할까요?

### Session (Stateful)
*   **방식:** 서버 메모리나 DB에 "세션 ID = 홍길동"이라고 적어둠. 클라이언트는 세션 ID만 쿠키로 들고 다님.
*   **장점:** 즉시 로그아웃(Kick) 가능. (서버에서 지우면 끝)
*   **단점:** 서버 메모리 차지. Scale-out 시 Redis 필요.

### Token (Stateless, JWT)
*   **방식:** "이 사람은 홍길동임(서명됨)"이라는 종이(토큰)를 클라이언트에게 줌. 서버는 아무것도 기억 안 함.
*   **장점:** 확장성 좋음.
*   **단점:** 토큰 탈취당하면 만료될 때까지 답 없음. (Blacklist 운영 필요)

**Authentik은 하이브리드입니다.**
*   Authentik 자체 로그인: **Session** 기반 (Redis 저장).
*   OAuth/OIDC 연동: **Token** 기반 (JWT 발급).

---

## 4. Federated Identity (연합 인증)

"우리 회사 아이디로 파트너사 포털에 로그인하고 싶다."

이때 등장하는 것이 **Federation**입니다. 신뢰 관계(Trust)를 맺은 두 도메인 간에 인증 정보를 교환하는 기술입니다.
*   **SAML, OIDC**가 바로 이 기술의 구현체입니다.
*   Authentik은 **IdP(신원 제공자)**가 될 수도 있고, 다른 Google/GitHub 등을 상위 IdP로 두는 **SP(서비스 제공자)**가 될 수도 있습니다. 이를 **Social Login** 또는 **Upstream Source**라고 합니다.

---

## 5. Summary

*   로그인 페이지가 뜨는 건 **AuthN**의 영역입니다.
*   "접근 권한이 없습니다" 페이지가 뜨는 건 **AuthZ**의 영역입니다.
*   Authentik은 **RBAC** (그룹 기반)과 **ABAC** (정책 기반)을 모두 지원하여 정교한 권한 제어가 가능합니다.

이 개념을 머릿속에 박고, Authentik의 Policy 엔진을 만지러 갑시다.
