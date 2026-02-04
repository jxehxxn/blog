---
layout: post
title: "IAM Mastery with Authentik: Week 11 - Auditing & Compliance"
---

보안 사고가 터졌습니다. 가장 먼저 하는 질문은 무엇일까요?
"누가 그랬어?", "언제 그랬어?", "어떻게 들어왔어?"

이 질문에 1분 안에 대답하지 못하면 보안 시스템이 없는 것과 같습니다.
Authentik은 모든 행위를 기록합니다. 이를 분석하고 모니터링하는 법을 배웁니다.

---

## 1. Events & Logs: The Black Box

Authentik의 Admin Interface -> **Events** 메뉴에서 모든 것을 볼 수 있습니다.

### 주요 이벤트 유형
*   `login`: 로그인 성공.
*   `login_failed`: 로그인 실패. (매우 중요 - Brute Force 공격 징후)
*   `authorize_application`: 특정 앱에 접근.
*   `model_created/updated/deleted`: 관리자가 설정을 변경함. (누가 정책을 몰래 바꿨나?)
*   `invitation_used`: 초대 링크 사용.

### 로그 상세 정보
로그를 클릭하면 JSON 형태의 상세 정보가 나옵니다.
*   `user`: 행위자.
*   `http_request`: IP, User-Agent.
*   `context`: 어떤 Flow, 어떤 Stage에서 발생했는지.

---

## 2. Notification & Alerting

로그만 쌓아두면 소용없습니다. 이상 징후가 보이면 알람이 울려야 합니다.
Authentik의 **Notification Transport** 기능을 사용합니다.

### 지원 채널
*   Slack / Discord Webhook
*   Email
*   Generic Webhook (OpsGenie, PagerDuty 연동용)

### Notification Rule
"이벤트 A가 발생하면 채널 B로 알려라."
1.  **Rule 생성:** `Severity`가 `Alert` 이상인 경우.
2.  **Mapping:** `login_failed` 이벤트가 발생하면.
3.  **Action:** Slack의 `#security-alerts` 채널로 메시지 전송.

---

## 3. SIEM Integration (Splunk, ELK)

엔터프라이즈 환경에서는 Authentik 로그만 따로 보지 않습니다. 방화벽 로그, OS 로그와 합쳐서 통합 분석(Correlation)을 해야 합니다.

Authentik은 로그를 외부로 내보내는 두 가지 방법을 제공합니다.
1.  **Syslog/Stdout:** 컨테이너 로그를 Fluentd/Logstash가 수집. (가장 일반적)
2.  **Webhooks:** 이벤트 발생 시 Splunk HEC(HTTP Event Collector)로 JSON 전송.

---

## 🛠️ Lab: Login Failure Alert

비밀번호 틀릴 때마다 Slack 알림이 오도록 설정해봅시다. (테스트용)

1.  **Transport 생성:**
    *   System -> Notification Transports -> Create **Slack/Discord Webhook**.
    *   여러분의 Slack Incoming Webhook URL 입력.
2.  **Rule 생성:**
    *   System -> Notification Rules -> Create.
    *   Name: "Login Failures".
    *   Severity: Warning.
3.  **Policy 바인딩 (중요):**
    *   Rule을 특정 그룹에게만 적용할지, 전체 적용할지 결정해야 합니다. (기본적으로 관리자 그룹에 적용됨)
4.  **Trigger:**
    *   로그아웃 후, 틀린 비밀번호를 입력해 봅니다.
    *   Slack이 울리나요?

---

## 📝 11주차 과제: Audit Report Analysis

**목표:** 지난 일주일간의 로그 데이터를 엑셀(CSV)로 추출하여 다음 질문에 답하는 리포트를 작성하세요. (가상 데이터라도 좋습니다)

1.  **Top 3 Users:** 가장 로그인을 많이 한 사용자 3명은?
2.  **Suspicious IPs:** 실패 횟수가 10회 이상인 IP 주소 목록은?
3.  **Policy Changes:** 지난주에 정책(Policy)을 수정하거나 삭제한 관리자 계정은 누구인가?

*힌트: Authentik은 API를 통해 이벤트 로그를 JSON으로 덤프할 수 있습니다.*

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Authentik에서 관리자가 실수로 사용자를 삭제했습니다. 이를 복구하거나 누가 했는지 찾기 위해 확인해야 할 메뉴는? (Admin Interface -> Events -> Logs)
2.  **Q2.** 로그의 무결성을 위해, 로그 서버는 어떤 보안 조치가 필요한가요? (Write-once, Read-many - 수정 불가능한 저장소)
3.  **Q3.** 로그인 실패 로그(`login_failed`)가 1초에 100건씩 찍히고 있습니다. 의심되는 공격 유형은? (Credential Stuffing / Brute Force Attack)

---

보안 관제까지 끝났습니다. 이제 여러분은 혼자가 아닙니다. Authentik이 지켜보고 있습니다.
다음 주, 대망의 **마지막 주차**입니다. 이 모든 시스템을 고가용성(HA) 아키텍처로 프로덕션에 배포하는 **Capstone Project**를 수행합니다.

**Watcher of the walls.**
