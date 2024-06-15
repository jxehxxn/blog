---
layout: post
title: "[Splunk 12주 완성] 2주차: 로깅 파이프라인과 데이터 온보딩 (심화 마스터 클래스)"
date: 2024-03-08 09:00:00 +0900
categories: splunk
---

# [Lecture Note] 2주차: 데이터 온보딩과 파이프라인의 심층 이해

본 강의는 Splunk의 핵심인 데이터 온보딩(Data Onboarding)과 로깅 파이프라인의 내부 동작 원리를 3시간에 걸쳐 심층적으로 다루는 마스터 클래스입니다. 단순히 설정을 복사해서 붙여넣는 수준을 넘어, 데이터가 소스에서 인덱서까지 흐르는 과정에서 발생하는 내부 큐(Queue)의 동작, 메모리 관리, 그리고 대규모 환경에서의 트러블슈팅 기법을 완벽하게 이해하는 것을 목표로 합니다.

---

## Module 1: 로그 생성 (The Source) - 데이터는 어디서 오는가?

많은 입문자가 로그가 이미 서버에 존재한다고 가정합니다. 하지만 실무에서는 로그가 Splunk에서 분석하기 좋은 형태로 생성되도록 소스 시스템을 먼저 설정해야 합니다.

### 1. Web Server: Nginx/Apache JSON 로깅 설정
기본적인 텍스트 로그는 파싱이 어렵습니다. 처음부터 JSON 형태로 로그를 생성하면 Splunk에서 별도의 정규표현식 없이도 필드를 자동으로 인식합니다.

**Nginx 설정 (`/etc/nginx/nginx.conf`):**
```nginx
http {
    log_format json_analytics escape=json
        '{'
            '"time_local":"$time_local",'
            '"remote_addr":"$remote_addr",'
            '"remote_user":"$remote_user",'
            '"request":"$request",'
            '"status": "$status",'
            '"body_bytes_sent":"$body_bytes_sent",'
            '"request_time":"$request_time",'
            '"http_referrer":"$http_referer",'
            '"http_user_agent":"$http_user_agent"'
        '}';

    access_log /var/log/nginx/access.json.log json_analytics;
}
```

**Apache 설정 (`/etc/httpd/conf/httpd.conf`):**
```apache
LogFormat "{ \
  \"time\":\"%t\", \
  \"remote_ip\":\"%a\", \
  \"host\":\"%V\", \
  \"request\":\"%U\", \
  \"query\":\"%q\", \
  \"method\":\"%m\", \
  \"status\":\"%>s\", \
  \"userAgent\":\"%{User-agent}i\", \
  \"referer\":\"%{Referer}i\" \
}" json_log

CustomLog "logs/access_json.log" json_log
```

### 2. Linux 시스템 로그: rsyslog
리눅스 커널이나 시스템 서비스 로그는 `rsyslog`를 통해 관리됩니다. 이를 로컬 파일로 저장하거나 Splunk로 직접 전송할 수 있습니다.

**rsyslog 설정 (`/etc/rsyslog.d/splunk.conf`):**
```conf
# 모든 로그를 특정 파일에 저장
*.* /var/log/all_system_events.log

# 또는 특정 포트로 직접 전송 (UDP)
*.* @10.1.1.10:514
```

### 3. Windows Event Forwarding (WEF)
윈도우 환경에서는 각 서버의 이벤트 로그를 수집 서버(Collector)로 모은 뒤, 그곳에서 Splunk Universal Forwarder를 통해 전송하는 방식이 효율적입니다. `Event Viewer`에서 구독(Subscription)을 설정하여 보안, 시스템, 애플리케이션 로그를 중앙 집중화합니다.

---

## Module 2: 로그 수집 (The Collection) - Universal Forwarder (UF)

로그가 생성되었다면 이제 Splunk로 안전하게 옮겨야 합니다. 이때 사용하는 도구가 Universal Forwarder(UF)입니다.

### 1. UF 설치 및 기본 연결
UF는 리소스를 최소한으로 사용하는 경량 에이전트입니다.

```bash
# 설치 및 실행
tar xvzf splunkforwarder-9.x.x.tgz -C /opt/
/opt/splunkforwarder/bin/splunk start --accept-license

# 인덱서(데이터 저장소) 주소 등록
/opt/splunkforwarder/bin/splunk add forward-server 10.1.1.10:9997
```

### 2. inputs.conf: 무엇을 읽을 것인가?
Nginx의 JSON 로그를 읽도록 설정해 보겠습니다.

**`/opt/splunkforwarder/etc/system/local/inputs.conf`:**
```ini
[monitor:///var/log/nginx/access.json.log]
index = web_logs
sourcetype = nginx:json
disabled = 0
```

### 3. outputs.conf: 어디로 보낼 것인가?
데이터 전송의 안정성을 위해 `useACK` 옵션을 활성화합니다.

**`/opt/splunkforwarder/etc/system/local/outputs.conf`:**
```ini
[tcpout:default-autolb-group]
server = 10.1.1.10:9997
useACK = true
```

---

## Module 3: 웹 로그 그 너머 (Diverse Data Sources)

실무에서는 웹 로그 외에도 다양한 데이터를 다룹니다.

### 1. Database Audit Logs
데이터베이스의 쿼리 이력이나 보안 감사 로그는 매우 중요합니다.
- **MySQL:** `server-audit-log`를 활성화하여 파일로 저장한 뒤 UF로 수집합니다.
- **PostgreSQL:** `postgresql.conf`에서 `logging_collector = on`과 `log_statement = 'all'`을 설정합니다.

### 2. Network Logs (Firewall/Switch)
네트워크 장비는 에이전트를 설치할 수 없습니다. 따라서 장비가 내보내는 Syslog를 Splunk가 직접 받거나, 중간에 Syslog-ng 서버를 두어 파일로 저장한 뒤 수집합니다.

### 3. Application Logs (Log4j/Logback)
자바 애플리케이션은 `Log4j`를 많이 사용합니다.
- **File Appender:** 로그를 파일로 쓰고 UF가 읽는 방식 (가장 권장됨).
- **Socket Appender:** 네트워크를 통해 직접 전송하는 방식 (애플리케이션 성능에 영향을 줄 수 있음).

---

## Module 4: Zero-to-Hero 로깅 파이프라인 구축 워크플로우

실제 환경에서 파이프라인을 구축하는 순서는 다음과 같습니다.

1.  **요구사항 분석:** 어떤 필드가 필요한지, 보관 주기(Retention)는 얼마인지 결정합니다.
2.  **소스 시스템 설정:** Nginx나 DB에서 로그 생성을 활성화하고 포맷을 JSON으로 맞춥니다.
3.  **인덱스 생성:** Splunk 서버에서 데이터를 담을 그릇인 `Index`를 미리 만듭니다.
4.  **UF 설치 및 설정:** 대상 서버에 UF를 깔고 `inputs.conf`와 `outputs.conf`를 작성합니다.
5.  **데이터 검증:** Splunk Search Head에서 `index=web_logs` 쿼리로 데이터가 잘 들어오는지 확인합니다.
6.  **파싱 최적화:** 타임스탬프가 정확한지, 멀티라인 로그가 깨지지 않는지 점검합니다.

---

## Module 5: Splunk 데이터 파이프라인의 4단계 세그먼트

데이터가 소스에서 인덱서까지 흐르는 내부 과정을 이해해야 트러블슈팅이 가능합니다.

### 1. Input Segment (입력 단계)
- **수행 위치:** Universal Forwarder (UF)
- **동작:** 데이터 소스로부터 원시 데이터를 읽어 64KB 블록으로 쪼갭니다.
- **핵심:** `host`, `source`, `sourcetype` 메타데이터가 여기서 처음 붙습니다.

### 2. Parsing Segment (파싱 단계)
- **수행 위치:** Heavy Forwarder (HF) 또는 Indexer
- **동작:** 개별 이벤트로 분리하고 타임스탬프를 추출합니다.
- **설정:** `LINE_BREAKER`, `TIME_FORMAT` 등이 여기서 적용됩니다.

### 3. Indexing Segment (인덱싱 단계)
- **수행 위치:** Indexer
- **동작:** 데이터를 디스크의 버킷(Bucket)에 쓰고 검색 인덱스를 생성합니다.
- **특징:** 이 단계가 끝나면 원본 데이터는 수정할 수 없습니다.

### 4. Search Segment (검색 단계)
- **수행 위치:** Search Head
- **동작:** 사용자의 SPL 쿼리에 따라 인덱서에서 데이터를 가져와 시각화합니다.

---

## Module 6: 큐(Queue) 메커니즘과 성능 최적화

Splunk는 데이터 유실을 막기 위해 내부적으로 큐를 사용합니다.

- **Memory Queues:** 기본적으로 모든 큐는 메모리에 상주합니다 (512KB).
- **Persistent Queues:** 네트워크 장애 시 데이터를 잃지 않도록 디스크 공간을 큐로 사용합니다.
  ```ini
  [monitor:///var/log/app.log]
  persistentQueueSize = 5GB
  ```
- **Backpressure:** 인덱서가 바쁘면 포워더의 큐가 가득 차고, 결국 데이터 수집이 멈춥니다. 이는 데이터 유실을 방지하는 정상적인 동작입니다.

---

## Module 7: 3시간 강의 요약 및 과제

### 핵심 요약
1.  로그는 소스 시스템(Nginx, DB 등)에서 먼저 올바르게 생성되어야 합니다.
2.  JSON 포맷은 Splunk 온보딩의 치트키입니다.
3.  UF는 `inputs.conf`로 읽고 `outputs.conf`로 보냅니다.
4.  데이터는 Input -> Parsing -> Indexing -> Search의 단계를 거칩니다.

### 실습 과제
- 자신의 로컬 환경에 Nginx를 설치하고 JSON 로그를 생성하도록 설정하세요.
- UF를 통해 해당 로그를 Splunk로 전송하고, 특정 필드(예: status)로 통계를 내보세요.
- `outputs.conf`에서 `useACK = true`를 설정한 뒤 인덱서를 끄고 UF의 로그 변화를 관찰하세요.

---
**Next Week:** 3주차에는 수집된 데이터를 요리하는 SPL(Search Processing Language) 기초와 통계 분석에 대해 배웁니다.
