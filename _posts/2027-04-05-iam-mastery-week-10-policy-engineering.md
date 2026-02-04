---
layout: post
title: "IAM Mastery with Authentik: Week 10 - Advanced Policy Engineering (Python)"
---

Authentik UI에서 클릭만으로 할 수 있는 건 한계가 있습니다.
"입사일이 3개월 미만인 인턴은 주말에 VPN 접속 불가, 단 팀장 승인이 있으면 허용."
이런 복잡한 로직은 **Python Expression Policy**로 구현해야 합니다.

Authentik은 샌드박스화된 Python 환경을 제공하여, 인증/인가 로직을 프로그래밍할 수 있게 해줍니다.

---

## 1. The Context Object: `request`

정책 코드 안에서 가장 중요한 객체는 `request`입니다.

*   `request.user`: 현재 로그인 시도 중인 사용자 객체.
    *   `request.user.username`
    *   `request.user.email`
    *   `request.user.attributes['phone']` (커스텀 속성)
    *   `request.user.group_names` (그룹 목록)
*   `request.http_request`: HTTP 요청 정보.
    *   `request.http_request.META['REMOTE_ADDR']` (IP 주소)
    *   `request.http_request.headers['User-Agent']` (브라우저 정보)
*   `request.context`: Flow 내에서 전달되는 데이터.

---

## 2. Writing Policies

정책은 `True`(통과) 또는 `False`(거부/실패)를 반환해야 합니다.

### 예제 1: 업무 시간 제한
```python
from datetime import datetime

now = datetime.now()
# 월(0) ~ 금(4)이고, 9시~18시 사이
is_weekday = 0 <= now.weekday() <= 4
is_work_time = 9 <= now.hour < 18

if is_weekday and is_work_time:
    return True

ak_message("업무 시간 외에는 접속할 수 없습니다.") # 사용자에게 보여줄 메시지
return False
```

### 예제 2: 특정 IP 대역 허용 (CIDR)
```python
from netaddr import IPNetwork, IPAddress

client_ip = IPAddress(request.http_request.META.get('REMOTE_ADDR'))
allowed_network = IPNetwork('192.168.0.0/16')

if client_ip in allowed_network:
    return True
return False
```

---

## 3. Dynamic Application Access

이 정책을 어디에 쓸까요?
1.  **Flow:** 로그인 자체를 막을 때.
2.  **Application:** 특정 앱 아이콘을 숨기거나 접근을 막을 때.

Application 설정의 **Policy Engine Mode**가 중요합니다.
*   **ANY:** 연결된 정책 중 하나만 통과하면 OK.
*   **ALL:** 연결된 모든 정책을 통과해야 OK.

---

## 🛠️ Lab: The "Manager Only" Button

특정 앱(Application)에 정책을 걸어서, 일반 사용자에게는 숨기고 관리자에게만 보이게 합니다.

1.  **Policy 생성:**
    ```python
    return "Managers" in request.user.group_names
    ```
2.  **Application 설정:**
    *   `Super Secret Admin App` 편집.
    *   **Policies** 탭에서 위 정책을 바인딩.
3.  **테스트:**
    *   일반 유저로 로그인 -> 앱 목록에 안 보임.
    *   Managers 그룹 유저로 로그인 -> 앱 목록에 보임.

---

## 📝 10주차 과제: Conditional MFA Policy

**목표:** "사내망(10.0.0.0/8)에서 접속하면 MFA를 건너뛰고, 외부망에서 접속할 때만 MFA를 요구"하는 정책을 작성하세요.

1.  **IP 체크 정책:** 접속 IP가 사내망인지 확인하는 Python Policy.
2.  **MFA Flow 수정:** MFA Validation Stage에 이 정책을 바인딩합니다.
3.  **Negate(반전) 여부:** "사내망이면 True" 정책을 만들었다면, MFA Stage에서는 "정책이 False일 때(외부망일 때) 실행"하도록 설정해야 합니다. (Policy Binding 설정에 `Negate` 옵션이 있습니다.)

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Authentik Expression Policy에서 사용자에게 거부 사유 메시지를 띄우기 위해 사용하는 함수는? (`ak_message("...")`)
2.  **Q2.** 정책 코드 내에서 `import os`나 `import subprocess`를 사용할 수 있나요? (아니요. 보안을 위해 샌드박스 처리되어 있어 제한된 라이브러리만 사용 가능합니다.)
3.  **Q3.** Application의 Policy Mode가 `ALL`일 때, 정책 3개 중 하나라도 False를 반환하면 결과는? (접근 거부 - False)

---

이제 여러분은 코드로 보안을 제어하는 엔지니어가 되었습니다.
다음 주, "누가 언제 들어왔는지" 감시하는 **Auditing & Compliance**를 배웁니다. 로그는 거짓말을 하지 않습니다.

**Code is Law.**
