---
layout: post
title: "[Splunk 12주 완성] 6주차: 실시간 탐지와 자동화 (SOAR의 기초)"
date: 2024-04-05 09:00:00 +0900
categories: splunk
tags: [splunk, splunk-alert, soar, security-automation]
---

보안 관제 센터(SOC)에서 가장 무서운 적은 해커가 아닙니다. 바로 알람 피로(Alert Fatigue)입니다. 하루에 수천 개씩 쏟아지는 알람 속에서 진짜 위협을 찾아내는 일은 모래사장에서 바늘 찾기보다 어렵습니다. 6주차인 이번 시간에는 단순한 검색을 넘어 분석가를 지치지 않게 하면서도 정확한 탐지를 수행하는 방법을 알아보겠습니다.

### 1. 탐지 로직의 심층 분석: 스케줄 vs 실시간

Splunk에서 알람을 만드는 과정은 간단합니다. 잘 짜인 SPL 검색 결과를 저장할 때 Alert 타입을 선택하면 됩니다. 하지만 실무에서는 여기서부터 고민이 시작됩니다. 알람은 크게 두 가지 방식으로 작동합니다.

#### 스케줄(Scheduled) 알람과 Cron 표현식
대부분의 보안 탐지는 시스템 부하를 줄이기 위해 스케줄 방식을 권장합니다. 5분마다 혹은 1시간마다 정해진 주기에 따라 검색을 수행합니다. 이때 사용하는 것이 Cron 표현식입니다.

Cron은 다섯 개의 필드로 구성됩니다.
- 분 (0-59)
- 시 (0-23)
- 일 (1-31)
- 월 (1-12)
- 요일 (0-6, 0은 일요일)

예를 들어 `*/5 * * * *`는 5분마다 검색을 실행하라는 뜻입니다. `0 0 * * *`는 매일 자정에 실행합니다. 스케줄 알람은 인덱서의 자원을 예측 가능하게 사용하므로 대규모 환경에서 안정적입니다.

#### 실시간(Real-time) 알람의 함정
이 방식은 조건이 충족되는 즉시 발생합니다. CPU 사용량이 99%를 찍는 순간 바로 알아야 하는 상황에 적합합니다. 하지만 실시간 알람은 인덱서에 지속적인 검색 부하를 줍니다. 검색이 끝나지 않고 계속 떠 있는 상태이기 때문입니다.

작동 원리는 슬라이딩 윈도우(Sliding Window) 방식을 따릅니다. 최근 5분간의 데이터를 계속해서 감시하며 조건이 맞는 순간 트리거합니다. 100개의 실시간 알람을 돌리면 100개의 검색 프로세스가 서버 자원을 계속 점유한다는 점을 명심해야 합니다. 꼭 필요한 경우가 아니라면 1분 단위 스케줄 알람으로 대체하는 것이 좋습니다.

### 2. 억제(Throttling)와 임계값 설정 전략

단순히 로그가 찍히면 알려달라고 설정하면 지옥을 맛보게 됩니다. 특정 IP에서 로그인 실패가 1번 발생했다고 알람을 띄우면 비밀번호를 한 번 틀린 모든 직원이 보안 위협으로 간주됩니다.

#### 임계값(Threshold) 설정
임계값은 얼마나 많이 발생했을 때 알람을 보낼지 결정합니다.
- **결과 개수가 0보다 클 때:** 단 한 번의 발생도 놓치지 않아야 할 때 (예: 관리자 계정 삭제)
- **결과 개수가 N보다 클 때:** 일정 횟수 이상 반복될 때 (예: 5분간 로그인 실패 10회 이상)

#### 억제(Throttling)의 이해
더 중요한 기술은 억제입니다. 한 번 알람이 발생했다면 동일한 이슈에 대해서는 일정 시간 동안 추가 알람을 보내지 않도록 설정하는 기능입니다.

설정 시 두 가지 옵션이 중요합니다.
1. **필드 값에 따른 억제:** 특정 필드(예: src_ip)를 기준으로 억제합니다. A라는 IP에서 공격이 들어와 알람이 떴다면 30분 동안 A IP에 대해서는 조용히 하고 B IP에서 들어오는 공격은 새로 알람을 띄웁니다.
2. **전체 알람 억제:** 어떤 결과가 나오든 첫 알람 이후 일정 시간 동안 모든 알람을 막습니다.

서버 한 대가 죽어서 1초에 100개의 에러 로그를 뿜어낼 때 휴대폰으로 100개의 문자가 오게 해서는 안 됩니다. 첫 알람 이후 30분간은 조용히 하도록 설정하는 것이 분석가의 정신 건강에 이롭습니다.

### 3. 1시간 집중 실습: Python을 이용한 커스텀 웹훅 연동

Splunk 안에서만 알람을 보는 시대는 지났습니다. 팀원들이 가장 많이 머무는 곳으로 정보를 보내야 합니다. 이번 실습에서는 Python을 사용해 Splunk 알람 데이터를 Jira와 Slack으로 동시에 전송하는 커스텀 액션을 만들어 보겠습니다.

#### Step 1: Jira 티켓 생성 페이로드 (JSON)
Jira API에 전달할 데이터 구조입니다. 프로젝트 키와 이슈 유형, 요약 내용을 포함합니다.

```json
{
    "fields": {
        "project": {
            "key": "SEC"
        },
        "summary": "Splunk Alert: Brute Force Detected from 192.168.1.10",
        "description": "Multiple failed login attempts detected for user: admin. Please investigate immediately.",
        "issuetype": {
            "name": "Incident"
        },
        "priority": {
            "name": "High"
        }
    }
}
```

#### Step 2: Slack 메시지 페이로드 (JSON)
Slack의 Incoming Webhook을 위한 구조입니다. 가독성을 위해 Attachment 기능을 사용합니다.

```json
{
    "text": "🚨 *Splunk Security Alert*",
    "attachments": [
        {
            "color": "#FF0000",
            "blocks": [
                {
                    "type": "section",
                    "text": {
                        "type": "mrkdwn",
                        "text": "*Search Name:* Brute Force Attack\n*Severity:* High\n*Source IP:* 192.168.1.10"
                    }
                },
                {
                    "type": "actions",
                    "elements": [
                        {
                            "type": "button",
                            "text": {
                                "type": "plain_text",
                                "text": "View in Splunk"
                            },
                            "url": "https://splunk.internal/app/search/..."
                        }
                    ]
                }
            ]
        }
    ]
}
```

#### Step 3: Python 통합 스크립트
Splunk의 `$SPLUNK_HOME/bin/scripts` 경로에 저장할 스크립트입니다.

```python
import json
import requests
import sys

def send_to_jira(summary, description):
    url = "https://your-jira.atlassian.net/rest/api/2/issue"
    auth = ("user@example.com", "your-api-token")
    headers = {"Content-Type": "application/json"}
    payload = {
        "fields": {
            "project": {"key": "SEC"},
            "summary": summary,
            "description": description,
            "issuetype": {"name": "Incident"}
        }
    }
    requests.post(url, data=json.dumps(payload), headers=headers, auth=auth)

def send_to_slack(text, fields):
    url = "https://hooks.slack.com/services/T000/B000/XXXX"
    payload = {
        "text": text,
        "attachments": [{"color": "#eb4034", "fields": fields}]
    }
    requests.post(url, data=json.dumps(payload), headers={"Content-Type": "application/json"})

if __name__ == "__main__":
    # Splunk는 스크립트 실행 시 첫 번째 인자로 결과 파일 경로를 전달합니다.
    # 여기서는 간단한 예시를 위해 직접 데이터를 구성합니다.
    alert_name = "Brute Force Attack Detected"
    src_ip = "192.168.1.10"
    user = "admin_test"

    summary = f"[{alert_name}] Source: {src_ip}"
    description = f"User {user} failed to login multiple times from {src_ip}."

    # Jira 전송
    send_to_jira(summary, description)

    # Slack 전송
    slack_fields = [
        {"title": "Alert", "value": alert_name, "short": True},
        {"title": "User", "value": user, "short": True},
        {"title": "Source IP", "value": src_ip, "short": False}
    ]
    send_to_slack("🚨 Splunk Alert Triggered", slack_fields)
```

### 4. SOAR 플레이북: 피싱 대응 자동화 시나리오

알람이 발생한 뒤에 분석가가 수동으로 IP를 조회하고 평판 사이트를 뒤지고 방화벽 차단 정책을 넣는 과정은 너무 느립니다. 여기서 SOAR(Security Orchestration, Automation and Response)가 등장합니다. 피싱 메일이 신고되었을 때 SOAR가 수행하는 자동화된 논리 흐름(Playbook)을 살펴봅시다.

#### 피싱 대응 플레이북의 논리적 흐름
1. **트리거:** 사용자가 이메일 클라이언트에서 신고 버튼을 누르거나 이메일 보안 장비가 의심 메일을 격리합니다.
2. **아티팩트 추출:** 메일 본문에서 URL, 첨부 파일의 해시(Hash), 발신자 IP, 발신 도메인을 자동으로 추출합니다.
3. **평판 조회 (Enrichment):** URL은 VirusTotal 및 URLScan.io에서 조회하고 파일 해시는 내부 샌드박스에 투고합니다. 발신 IP는 Whois 및 블랙리스트 DB와 대조합니다.
4. **상관 분석:** 해당 메일을 받은 다른 수신자가 조직 내에 더 있는지 이메일 서버를 검색합니다.
5. **위험도 판별:** 하나 이상의 엔진에서 악성으로 판정되면 즉시 차단 단계로 진입합니다. 판정이 모호하면 분석가에게 Slack으로 승인 요청을 보냅니다.
6. **자동 대응 (Remediation):** 조직 내 모든 사용자의 편지함에서 해당 메일을 강제 삭제합니다. 방화벽 및 프록시 서버에서 악성 URL/IP를 차단하고 메일을 열어본 사용자의 계정을 일시 잠금합니다.
7. **종료:** Jira 티켓을 업데이트하고 분석 보고서를 자동 생성하여 종결합니다.

이런 일련의 과정을 자동화하면 분석가는 단순 반복 작업에서 벗어나 더 고도화된 위협 사냥(Threat Hunting)에 집중할 수 있습니다.

---

### 6주차 자가 진단 퀴즈

**문제 1:** 동일한 조건의 알람이 너무 자주 발생하여 분석가가 피로를 느낄 때 Splunk 알람 설정에서 가장 먼저 검토해야 하는 기능은 무엇인가요?
**문제 2:** 실시간(Real-time) 알람보다 스케줄(Scheduled) 알람을 권장하는 주된 이유는 무엇인가요?
**문제 3:** SOAR 플레이북에서 외부 API를 통해 데이터를 보강하는 단계를 무엇이라 부르나요?

---

### 정답 및 해설

**정답 1:** 억제(Throttling). 특정 시간 동안 중복 알람이 나가지 않도록 제어해야 합니다.
**정답 2:** 시스템 자원 관리. 실시간 검색은 인덱서에 지속적인 부하를 주지만 스케줄 검색은 정해진 시간에만 자원을 사용하므로 훨씬 안정적입니다.
**정답 3:** 인리치먼트(Enrichment). 단순한 로그 데이터에 외부 정보를 결합하여 맥락을 풍부하게 만드는 과정입니다.

---

이번 주차 내용을 통해 여러분의 SOC가 조금 더 조용해지길 바랍니다. 다음 주에는 데이터 모델과 가속화에 대해 다뤄보겠습니다.
