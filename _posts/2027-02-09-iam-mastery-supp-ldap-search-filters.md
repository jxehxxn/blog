---
layout: post
title: "IAM Mastery with Authentik: Supplement - LDAP Search Filters & Schema"
---

Week 2 강의에서 LDAP 필터를 맛보기만 했습니다. 하지만 실무에서 Authentik이나 기타 솔루션을 AD와 붙일 때, 가장 많이 삽질하는 곳이 바로 **"사용자 필터(User Filter)"**와 **"그룹 필터(Group Filter)"** 설정입니다.

이 보충 포스트에서는 LDAP 쿼리를 자유자재로 다루기 위한 심화 내용을 다룹니다.

---

## 1. LDAP Schema: 객체는 어떻게 생겼나?

RDBMS에 테이블 스키마가 있다면, LDAP에는 **ObjectClass**가 있습니다.

### ObjectClass
어떤 속성(Attribute)을 가질 수 있는지 정의한 템플릿입니다.
*   **top:** 모든 객체의 조상.
*   **person:** `sn`(성), `cn`(이름), `userPassword`를 가짐.
*   **organizationalPerson:** `title`, `street`, `company` 등 조직 정보 추가.
*   **inetOrgPerson:** `mail`, `mobile`, `uid` 등 인터넷 정보 추가. (가장 많이 씀)
*   **user (AD):** 윈도우 사용자 전용 속성들 (`sAMAccountName` 등) 포함.

Authentik에서 사용자를 생성할 때 "어떤 ObjectClass를 쓸 거냐" 묻는 설정이 있습니다. 보통 `inetOrgPerson`을 씁니다.

---

## 2. Advanced Search Filters

### Wildcards
*   `*`: 모든 것. SQL의 `%`와 같습니다.
*   `(cn=J*)`: J로 시작하는 모든 이름.
*   `(mail=*@example.com)`: 특정 도메인 메일 사용자.

### Nested Logic (중첩 논리)
복잡해 보이지만 괄호만 잘 따라가면 됩니다.
`(& (condition1) (| (condition2) (condition3) ) )`
-> `condition1`은 무조건 만족해야 하고(AND), `condition2`나 `3` 중 하나만 만족하면 됨(OR).

**실무 예제 (Jira 연동 시):**
"jira-users 그룹에 속해 있거나, jira-admins 그룹에 속해 있는 활성 사용자만 로그인시키고 싶다."

```ldap
(&
  (objectClass=user)
  (!(userAccountControl:1.2.840.113556.1.4.803:=2))
  (|
    (memberOf=cn=jira-users,ou=Groups,dc=example,dc=com)
    (memberOf=cn=jira-admins,ou=Groups,dc=example,dc=com)
  )
)
```

---

## 3. The Bitwise Operator (AD Magic)

Active Directory의 `userAccountControl` 속성은 정수 하나에 여러 상태를 비트 플래그로 저장합니다.
*   512: Normal Account
*   2: Disabled Account
*   65536: Password Never Expires

이걸 검색하려면 특수한 확장 연산자(`:1.2.840.113556.1.4.803:`)를 써야 합니다. 이를 **LDAP_MATCHING_RULE_BIT_AND**라고 부릅니다.

*   **계정이 비활성화(2)된 사람 찾기:**
    `(userAccountControl:1.2.840.113556.1.4.803:=2)`

*   **계정이 활성화된 사람만 찾기 (NOT 비활성화):**
    `(!(userAccountControl:1.2.840.113556.1.4.803:=2))`

이 구문은 메모장에 저장해두고 평생 복사해서 쓰세요. 외우는 사람 없습니다.

---

## 4. Operational Attributes

`ls` 명령어로 파일 볼 때 숨겨진 파일이 있듯이, LDAP에도 조회 요청을 명시적으로 하지 않으면 안 보이는 속성들이 있습니다.

*   `createTimestamp`: 생성 시간
*   `modifyTimestamp`: 수정 시간
*   `nsUniqueId` / `objectGUID`: 고유 ID
*   `memberOf`: (일부 LDAP 서버에서는 요청해야 줌)

Authentik에서 매핑이 잘 안될 때, 속성 이름이 틀렸거나 이 Operational Attribute 문제일 수 있습니다.

---

## 5. Summary Checklist

Authentik에서 LDAP Source를 설정할 때 다음 체크리스트를 확인하세요.

1.  [ ] **Bind DN/Password:** 읽기 권한이 있는 서비스 계정인가?
2.  [ ] **Base DN:** 검색을 시작할 위치가 너무 상위(`dc=com`)라서 느리지 않은가? `ou=Users`로 좁힐 수 없는가?
3.  [ ] **User Filter:** `(objectClass=user)`만 쓰면 컴퓨터 객체도 검색될 수 있다. `(&(objectClass=user)(objectCategory=person))`을 써라.
4.  [ ] **Attributes:** 이메일 주소를 `mail`에서 가져올지 `userPrincipalName`에서 가져올지 결정했는가?

이 보충 자료가 여러분의 삽질 시간을 10시간 정도 줄여줄 겁니다.
