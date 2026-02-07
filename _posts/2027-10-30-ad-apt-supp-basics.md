---
layout: post
title: "Offensive AD & APT Mastery: Supplement - AD Fundamentals: The Absolute Basics"
---

반갑습니다.
공격 기술을 배우기 전, 잠시 숨을 고르고 **"기본기"**를 다질 시간입니다.

많은 분들이 `Mimikatz` 사용법은 알지만, 정작 "Kerberos가 뭐에요?"라고 물으면 "어... 인증하는 거요?"라고 얼버무립니다. 원리를 모르면 응용할 수 없고, 막히면 뚫을 수 없습니다.

오늘 우리는 **Active Directory, LDAP, NTLM, Kerberos**라는 4가지 핵심 개념을, IT를 전혀 모르는 사람도 이해할 수 있는 **비유(Analogy)**와 **실제 패킷 흐름**으로 완벽하게 해부해 봅니다.

---

## 1. Active Directory (AD): "스마트 빌딩 시스템"

### 개념 (Concept)
AD는 마이크로소프트가 만든 **"중앙 관리 시스템"**입니다.
회사에 직원(User)이 1000명, PC(Computer)가 1000대, 프린터가 50대 있습니다.
이걸 엑셀 파일 하나로 관리할 수 있을까요? 누군가 퇴사하면 모든 PC에서 그 사람 계정을 지워야 할까요?

그래서 AD가 탄생했습니다. **"모든 자원을 중앙(DC)에서 관리하자."**

### 비유 (Analogy)
AD는 거대한 **"스마트 빌딩 통합 관제 시스템"**입니다.

*   **Domain Controller (DC):** 빌딩 관리실(서버). 모든 권한을 가짐.
*   **User/Computer Object:** 입주민(직원)과 가구(PC).
*   **Group Policy (GPO):** 빌딩 규칙. "모든 사무실 에어컨은 24도로 고정해라", "모든 문은 6시에 잠가라". 관리실에서 버튼 하나 누르면 전 빌딩에 적용됩니다.
*   **Domain:** 빌딩 그 자체. (`samsung.com`, `google.local`)

### 실습 (Hands-on)
윈도우 서버(DC)에서 `Active Directory Users and Computers` (dsa.msc)를 열어보세요.
폴더(OU) 안에 사람 아이콘(User)과 컴퓨터 아이콘(Computer)이 보이죠? 그게 전부입니다.

---

## 2. LDAP: "도서관 사서에게 말하는 법"

### 개념 (Concept)
AD는 거대한 데이터베이스입니다. 여기서 "김철수 대리" 정보를 찾고 싶습니다.
이때 사용하는 **"검색 언어(Protocol)"**가 바로 **LDAP (Lightweight Directory Access Protocol)**입니다.

"SQL이 RDBMS의 언어라면, LDAP은 AD의 언어입니다."

### 구조 (The Tree)
LDAP은 폴더 트리 구조입니다. 주소가 아주 깁니다.

*   **DN (Distinguished Name):** 전체 주소.
    *   `CN=Cheolsoo,OU=Sales,DC=corp,DC=local`
    *   해석: `corp.local` 도메인 -> `Sales` 부서 폴더 -> `Cheolsoo`라는 사람.

### 비유 (Analogy)
도서관에서 책을 찾을 때 사서에게 이렇게 말하는 것과 같습니다.
"**한국 십진분류표(Protocol)**에 따라, **800번대(OU)** 서가에 있는, **'해킹'이라는 제목(CN)**을 가진 책을 찾아주세요."

### 실습 (Hands-on)
파워셸이나 터미널에서 다음 명령어를 쳐보세요. (도메인 PC에서)

```powershell
# "나(Administrator)"의 정보를 LDAP 언어로 물어보기
dsquery user -name administrator
```
**결과:** `"CN=Administrator,CN=Users,DC=corp,DC=local"`

---

## 3. NTLM: "스무고개 암호 대기" (구식)

### 개념 (Concept)
**NTLM (New Technology LAN Manager)**은 아주 오래된 인증 방식입니다.
**Challenge-Response** 방식을 사용합니다.

### 비유 (Analogy)
첩보 영화의 접선 장면을 상상해 보세요.

1.  **Client (나):** (서버에게 다가가며) "나 요원 A다. 문 열어."
2.  **Server (서버):** "못 믿겠는데? 자, 내가 내는 문제(Challenge / 난수)를 풀어봐. **'12345'**"
3.  **Client:** (내 비밀번호로 '12345'를 암호화함) "정답은 **'Xy9zQ'**다." (Response)
4.  **Server:** (자기 메모장에 있는 요원 A 비번으로 똑같이 계산해봄) "어? 답이 맞네? 들어와."

### 치명적 단점 (Vulnerability)
서버가 진짜 서버인지 확인을 안 합니다.
옆에 있던 해커가 "내가 서버야! 나한테 답해봐!"라고 끼어들면(MITM), 클라이언트는 해커에게 암호화된 답을 줍니다. 해커는 이걸 다른 서버에 던져서 문을 엽니다. (**NTLM Relay**)

---

## 4. Kerberos: "놀이공원 자유이용권" (최신)

### 개념 (Concept)
NTLM의 문제를 해결하기 위해 **"신뢰할 수 있는 제3자(KDC)"**를 도입했습니다.
비밀번호를 네트워크에 뿌리는 게 아니라, **티켓(Ticket)**을 보여주고 통과합니다.

### 비유 (Analogy) - **가장 중요!**
여러분이 롯데월드(Domain)에 갔습니다.

1.  **매표소 (AS - Authentication Service):**
    *   여러분: "저 철수인데요(ID), 표 주세요." (신분증 제시)
    *   직원: "확인됐습니다. 여기 **자유이용권(TGT)** 받으세요."
    *   *특징: 이 자유이용권은 절대 잃어버리면 안 됩니다.*

2.  **티켓 교환소 (TGS - Ticket Granting Service):**
    *   여러분: "저 **바이킹(File Server)** 타고 싶어요." (자유이용권 제시)
    *   직원: "네, 여기 **바이킹 전용 탑승권(ST)** 입니다."

3.  **바이킹 입구 (Service):**
    *   여러분: (알바생에게) "탑승권(ST) 여기요."
    *   알바생: "네, 타세요."
    *   *특징: 알바생은 여러분의 신분증(비번)을 검사하지 않습니다. 티켓만 봅니다.*

### 공격 포인트 (Hacking Point)
*   **Golden Ticket:** 매표소의 **도장(krbtgt)**을 훔쳐서 내가 집에서 자유이용권을 찍어냅니다. (유효기간 100년, VIP 권한)
*   **Kerberoasting:** 바이킹 전용 탑승권(ST)을 달라고 한 뒤, 타지는 않고 집에 가져가서 **알바생의 비밀번호**를 알아냅니다. (티켓이 알바생 비번으로 암호화되어 있으니까)

---

## 5. Summary Table

| 구분 | NTLM (구식) | Kerberos (신식) |
| :--- | :--- | :--- |
| **방식** | 스무고개 (Challenge-Response) | 티켓 보여주기 (Ticket-based) |
| **신뢰** | 서로 의심함 | 매표소(KDC)를 믿음 |
| **속도** | 느림 (매번 확인) | 빠름 (티켓만 보면 됨) |
| **약점** | Relay 공격에 취약 | 티켓 탈취/위조에 취약 |
| **사용처** | IP 접속, 워크그룹 | 도메인 접속, AD 환경 |

이 4가지 개념이 머릿속에 잡혔다면, 이제 여러분은 **"해킹할 준비"**가 되었습니다.
Week 2부터 이 개념들이 어떻게 무기로 변하는지 보여드리겠습니다.
