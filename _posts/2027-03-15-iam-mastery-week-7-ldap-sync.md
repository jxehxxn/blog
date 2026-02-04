---
layout: post
title: "IAM Mastery with Authentik: Week 7 - Inbound Sync: Connecting LDAP/AD to Authentik"
---

Authentik을 설치하고 Flow도 이해했습니다. 이제 텅 빈 Authentik에 사용자 정보를 채워 넣을 시간입니다.
일일이 `Create User` 버튼을 누를 건가요? 1000명을요?

당연히 기존에 쓰던 **Active Directory (AD)**나 **LDAP** 서버에서 긁어와야 합니다.
이것을 **Inbound Sync**라고 합니다.

---

## 1. The Sync Architecture

Authentik은 LDAP 정보를 실시간으로 프록시하지 않습니다. 주기적으로 **동기화(Sync)**하여 내부 DB에 복사본을 만듭니다.

1.  **LDAP Source:** Authentik에게 "저기 가서 긁어와"라고 설정합니다.
2.  **Bind User:** AD를 읽을 수 있는 계정으로 로그인합니다.
3.  **Sync Task:** Worker가 주기적으로(예: 10분마다) 변경 사항을 체크합니다.
4.  **Property Mapping:** AD의 `sAMAccountName`을 Authentik의 `username`으로 변환합니다.

이 방식의 장점은 **"AD가 죽어도 로그인은 된다"**입니다. (캐싱된 정보 사용)

---

## 2. Setting up LDAP Source

Admin Interface -> Directory -> Federation & Social Login -> **Create LDAP Source**

### Connection Settings
*   **Server URI:** `ldap://ad.example.com` (보안을 위해 `ldaps://` 권장)
*   **Bind DN:** `cn=service_account,ou=Users,dc=example,dc=com`
*   **Base DN:** `dc=example,dc=com` (검색 시작 위치)

### Password Update
"사용자가 Authentik에서 비번을 바꾸면 AD에도 반영되나요?"
네, **Writeback** 설정을 켜면 됩니다. 하지만 권한 설정이 까다로우니 처음엔 Read-only로 시작하세요.

---

## 3. The Sync Machinery: Users & Groups

가장 중요한 건 **"누구를 데려올 것인가"**입니다. Week 2에서 배운 필터가 여기서 쓰입니다.

### User Matching
*   **Query:** `(&(objectClass=user)(objectCategory=person))`
*   **Unique Attribute:** `objectGUID` (필수! 이름은 바뀌어도 이건 안 바뀜)

### Group Matching
*   **Query:** `(objectClass=group)`
*   AD의 그룹 구조를 Authentik 그룹으로 그대로 가져올 수 있습니다. 이를 통해 "AD에서 'VPN-Users' 그룹에 넣으면 자동으로 VPN 접근 허용" 같은 자동화를 구현합니다.

---

## 4. Troubleshooting Sync

"Sync를 돌렸는데 아무도 안 생겨요."

1.  **Connectivity:** Worker 컨테이너에서 `ping ad.example.com` 해보세요. DNS 문제일 수 있습니다.
2.  **Bind DN Error:** ID/PW가 틀렸거나 권한이 없습니다.
3.  **ObjectGUID Warning:** 로그에 "ObjectGUID is missing"이라고 뜨면, `objectGUID`를 바이너리 그대로 읽어서 깨진 겁니다. Property Mapping에서 `binascii` 등을 써서 문자열로 변환해줘야 합니다. (Authentik 기본 매핑은 이를 처리하고 있습니다.)

---

## 🛠️ Lab: Syncing OpenLDAP

Week 2에서 만든 로컬 OpenLDAP을 Authentik에 연결해 봅니다.

1.  **LDAP Source 생성:**
    *   URI: `ldap://host.docker.internal:389` (Docker 내부 통신)
    *   Bind DN: `cn=admin,dc=example,dc=org`
    *   Base DN: `dc=example,dc=org`
2.  **Sync 실행:** 'Run Sync' 버튼을 누릅니다.
3.  **로그 확인:** System -> Event Logs에서 Sync 작업의 로그를 봅니다. "Created User: Hoon"이 보이나요?
4.  **로그인 테스트:** 시크릿 창을 열고 LDAP에 있던 계정(`Hoon`)과 비밀번호로 로그인이 되는지 확인합니다.

---

## 📝 7주차 과제: Attribute Mapping Customization

기본 설정은 `Name`과 `Email`만 가져옵니다.

**목표:** LDAP의 `telephoneNumber` 속성을 가져와서, Authentik 사용자의 `attributes.phone` 필드에 저장하도록 매핑을 수정하세요.

1.  **Customization -> Property Mappings**로 이동합니다.
2.  `LDAP Property Mapping`을 새로 만듭니다.
3.  LDAP Attribute: `telephoneNumber`
4.  Object Field: `attributes.phone` (Authentik은 커스텀 데이터를 `attributes`라는 JSON 필드에 저장합니다)
5.  이 매핑을 LDAP Source 설정에 추가하고 Sync를 다시 돌리세요.
6.  사용자 상세 정보에서 전화번호가 보이는지 확인하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Authentik이 LDAP 서버와 실시간으로 통신하지 않고 주기적으로 데이터를 가져오는 방식의 장점은? (LDAP 서버 장애 시에도 로그인 가능 - 캐싱 효과)
2.  **Q2.** AD의 사용자 패스워드를 Authentik이 평문으로 알 수 있나요? (아니요. 해시된 값도 못 가져옵니다. 로그인은 Authentik이 입력받은 비번으로 AD에 'Bind' 시도(Bind Check)를 해서 검증합니다.)
3.  **Q3.** LDAP Sync 도중 "Size Limit Exceeded" 에러가 났습니다. 원인은? (LDAP 서버가 한 번에 줄 수 있는 결과 수 - 보통 1000개 - 를 초과함. Paging 설정 필요.)

---

이제 사용자가 생겼습니다. 다음 주, 이 사용자들을 데리고 외부 애플리케이션(**GitHub, Slack, Jenkins**)에 로그인하는 **Outbound Auth**를 배웁니다. 드디어 SSO가 완성됩니다.

**Sync complete.**
