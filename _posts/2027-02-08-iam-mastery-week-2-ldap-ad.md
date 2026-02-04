---
layout: post
title: "IAM Mastery with Authentik: Week 2 - The Directory: LDAP & Active Directory Deep Dive"
---

지난주에 IAM의 기본 개념을 잡았습니다. 그렇다면 기업들은 그 수많은 직원 정보를 어디에 저장할까요? 엑셀? DB?

대부분의 기업은 **Directory Service**를 사용합니다. 그리고 그 표준 프로토콜이 **LDAP(Lightweight Directory Access Protocol)**입니다. Authentik을 비롯한 모든 IAM 솔루션의 시작점은 바로 이 LDAP/AD와의 연동입니다.

이걸 모르면 "연동이 안 돼요"라고 울기만 하게 됩니다. 뜯어봅시다.

---

## 1. What is LDAP?

LDAP은 **"계층 구조로 된, 읽기 최적화된 데이터베이스"**를 조회하는 프로토콜입니다.
RDBMS(SQL)가 엑셀 표라면, LDAP은 **윈도우 탐색기 폴더 구조(트리)**와 같습니다.

### The Tree Structure (DIT - Directory Information Tree)
*   **Root:** 보통 도메인 이름에서 따옵니다. (`dc=example,dc=com`)
*   **Container:** 폴더 같은 역할. (`ou=Users`, `ou=Groups`)
*   **Leaf:** 실제 사용자나 컴퓨터. (`cn=Alice`)

### Key Terminology (이거 모르면 대화 불가)
*   **DN (Distinguished Name):** 전체 경로(Full Path). 유니크한 주소.
    *   예: `cn=Alice,ou=Marketing,dc=example,dc=com`
*   **CN (Common Name):** 이름. (`Alice`)
*   **OU (Organizational Unit):** 조직 단위. (`Marketing`)
*   **DC (Domain Component):** 도메인 구성요소. (`example`, `com`)

---

## 2. Active Directory (AD): The Big Boss

Microsoft가 만든 Active Directory는 **LDAP + Kerberos + DNS** 등을 합친 거대한 제품입니다. 전 세계 기업의 90%가 쓴다고 봐도 무방합니다.

Authentik을 구축할 때 가장 먼저 하는 일이 "기존 AD의 사용자 정보를 끌어오는 것"입니다.

### AD의 특징
*   **ObjectGUID:** 사용자의 주민등록번호 같은 불변의 ID. (DN은 부서 이동하면 바뀌지만, GUID는 안 바뀜)
*   **sAMAccountName:** 윈도우 로그인 ID. (예: `alice`)
*   **userPrincipalName (UPN):** 이메일 형태의 ID. (예: `alice@example.com`)
*   **MemberOf:** 사용자가 속한 그룹 정보.

---

## 3. LDAP Search Filter (검색의 기술)

LDAP에 쿼리를 날릴 때는 SQL(`SELECT * FROM...`)이 아니라 **Prefix Notation**을 씁니다.

*   `(` `연산자` `조건1` `조건2` ... `)`

**예시:**
1.  **이름이 'Alice'인 사람:**
    `(cn=Alice)`
2.  **이름이 'Alice'이고(AND), 'Marketing' 부서인 사람:**
    `(&(cn=Alice)(ou=Marketing))`
3.  **관리자 그룹(Admins)에 속하거나(OR), 개발팀(Devs)에 속한 사람:**
    `(|(memberOf=cn=Admins...)(memberOf=cn=Devs...))`
4.  **비활성화되지 않은(NOT) 사용자:**
    `(!(userAccountControl:1.2.840.113556.1.4.803:=2))` (AD 전용 비트 연산)

---

## 🛠️ Lab: Exploring LDAP

Authentik을 설치하기 전에, 오픈소스 LDAP 서버로 연습해봅시다.

### 준비물
*   Docker
*   LDAP 클라이언트 (Apache Directory Studio 권장)

### 1. Test LDAP Server 띄우기
```bash
docker run -p 389:389 -p 636:636 \
  --name my-openldap \
  --detach osixia/openldap:1.5.0
```

### 2. 접속 정보
*   **Host:** `localhost`
*   **Bind DN (ID):** `cn=admin,dc=example,dc=org`
*   **Password:** `admin`

### 3. 실습
1.  Apache Directory Studio로 접속하세요.
2.  `ou=People`이라는 OU를 만드세요.
3.  그 밑에 `inetOrgPerson` 클래스로 `cn=Hoon`이라는 사용자를 만드세요.
4.  `sn`(성)과 `mail`(이메일) 속성을 채워 넣으세요.

---

## 📝 2주차 과제: LDAP Query Master

다음 조건에 맞는 LDAP 검색 필터(Search Filter)를 작성하여 제출하세요.

1.  이메일(`mail`)이 존재하고(`*`), 성(`sn`)이 'Kim'인 사용자.
2.  `HR` 그룹의 멤버이면서, 동시에 `Finance` 그룹의 멤버인 사용자.
3.  (심화) Active Directory에서, 계정이 잠겨있지 않고(Enabled), 비밀번호가 만료되지 않은 사용자. (힌트: `userAccountControl`)

---

## ✅ Self-Assessment Quiz

1.  **Q1.** LDAP에서 `cn=admin,dc=example,dc=com`과 같은 고유한 전체 경로를 부르는 용어는? (DN - Distinguished Name)
2.  **Q2.** LDAP 필터 `(&(objectClass=user)(!(sn=Smith)))`의 의미는? (사용자 객체이면서, 성이 Smith가 아닌 사람)
3.  **Q3.** Authentik이 AD와 연동할 때, 사용자를 식별하기 위해 DN보다 `ObjectGUID`를 사용하는 것이 권장되는 이유는? (DN은 부서 이동 등으로 변경될 수 있지만 GUID는 불변하기 때문)

---

다음 주, 우리는 신분증(Identity)을 가지고 다른 서비스에 로그인하는 악수(Handshake)의 기술, **OAuth 2.0과 OIDC**를 파헤칩니다.

**Know your Directory.**
