---
layout: post
title: "[Splunk Week 13] 데이터 파이프라인 아키텍처와 라이선스 최적화: 빅테크의 생존 전략"
date: 2024-05-24 09:00:00 +0900
categories: splunk optimization
---

빅테크 기업에서 보안 운영을 하다 보면 가장 먼저 마주하는 벽은 기술이 아니라 비용입니다. 하루에 수십 테라바이트의 로그가 쏟아지는 환경에서 모든 데이터를 스플렁크(Splunk)에 인덱싱하는 건 자살 행위나 다름없습니다. 라이선스 비용이 보안 예산을 전부 잡아먹기 때문입니다. 13주 차 보너스 모듈에서는 빅테크 SIEM 아키텍트들이 어떻게 데이터를 걸러내고 비용을 아끼는지 실무적인 관점에서 다뤄보겠습니다.

이 장은 총 3시간 분량의 강의 내용을 담고 있습니다. 데이터 파이프라인 설계부터 실전 랩까지, 라이선스 비용을 90% 이상 절감하는 핵심 기술을 마스터해 봅시다.

---

## 1교시: 데이터 파이프라인 아키텍처 (1시간)

현대적인 빅테크 환경에서 로그를 소스에서 SIEM으로 직접 보내는 방식은 더 이상 유효하지 않습니다. 데이터의 양은 기하급수적으로 늘어나는데 예산은 한정되어 있기 때문입니다. 이를 해결하기 위해 등장한 개념이 바로 **관측성 파이프라인(Observability Pipeline)**입니다.

### 1.1 관측성 파이프라인의 필요성

데이터 소스와 목적지 사이에 중간 계층을 두면 다음과 같은 이점이 있습니다.
- **라우팅**: 데이터를 중요도에 따라 다른 곳으로 보냅니다. (예: 실시간 분석은 스플렁크, 장기 보관은 S3)
- **필터링**: 불필요한 디버그 로그나 중복 데이터를 인덱싱 전에 제거합니다.
- **변환**: 개인정보(PII)를 마스킹하거나 데이터 형식을 최적화합니다.
- **완충**: 갑작스러운 트래픽 증가 시 데이터를 버퍼링하여 시스템 다운을 막습니다.

### 1.2 주요 솔루션 비교

#### Cribl Stream: 데이터의 스위스 아미 나이프
크리블(Cribl)은 현재 이 분야의 독보적인 리더입니다. 자바스크립트 기반의 강력한 엔진으로 데이터를 실시간으로 가공합니다.

**Cribl 아키텍처 다이어그램:**
```text
[ 데이터 소스 ] ----> [ Cribl Stream Worker Group ] ----> [ 목적지 ]
(Syslog, HTTP,        (Leader Node가 관리)             (Splunk, S3,
 Kafka, S3)                                            Elastic, Kafka)
                             |
                     [ Routing Rules ]
                     [ Pipelines (JS/Regex) ]
```

#### Apache Kafka: 확장 가능한 데이터 백본
카프카는 대량의 데이터를 안정적으로 받아주는 버퍼 역할을 합니다. 스플렁크가 점검 중이거나 느려져도 로그 유실을 막아줍니다.

**Kafka 중심 아키텍처:**
```text
[ 소스 ] -> [ Kafka Producer ] -> [ Kafka Cluster ] -> [ Consumer ] -> [ Splunk ]
                                     (Topics)            (Cribl)
                                                            |
                                                     [ S3 / Data Lake ]
```

#### Splunk Ingest Actions & Edge Processor
스플렁크의 자체 기능입니다. 인덱서나 헤비 포워더에서 데이터를 거를 수 있습니다. 최근에는 소스 근처에서 데이터를 처리하는 **Edge Processor**가 출시되어 크리블과 경쟁하고 있습니다.

### 1.3 실전: transforms.conf를 이용한 고급 라우팅

크리블 같은 별도 솔루션이 없다면 스플렁크의 `transforms.conf`를 마스터해야 합니다.

**시나리오: 로그 레벨에 따른 멀티 데스티네이션 라우팅**
`ERROR` 로그는 메인 인덱스로, `INFO` 로그는 요약 인덱스로 보내고, `DEBUG` 로그는 버리고 싶을 때의 설정입니다.

**props.conf**
```conf
[source::app_logs]
TRANSFORMS-routing = route_error, route_info, route_debug
```

**transforms.conf**
```conf
[route_error]
REGEX = severity=ERROR
DEST_KEY = _MetaData:Index
FORMAT = security_critical

[route_info]
REGEX = severity=INFO
DEST_KEY = _MetaData:Index
FORMAT = security_summary

[route_debug]
REGEX = severity=DEBUG
DEST_KEY = queue
FORMAT = nullQueue
```

**시나리오: 복잡한 개인정보 마스킹 (Lookahead 활용)**
카드 번호의 앞부분은 가리고 마지막 4자리만 남기고 싶을 때 사용합니다.

**transforms.conf**
```conf
[mask_cc]
REGEX = (?<prefix>\d{4}-\d{4}-\d{4}-)(?<suffix>\d{4})
FORMAT = XXXX-XXXX-XXXX-$suffix
DEST_KEY = _raw
```

---

## 2교시: 실전 랩 - 노이즈 XML 로그를 메트릭으로 변환 (1시간)

윈도우 이벤트 로그(XML 형식)는 용량이 매우 큽니다. 성공한 로그인(EventCode 4624) 하나가 2~3KB를 차지하기도 합니다. 이런 로그가 하루에 수백만 건 쌓이면 라이선스 비용은 감당할 수 없게 됩니다.

### 2.1 이벤트에서 메트릭(Metrics)으로의 전환

모든 원본 로그를 보관할 필요는 없습니다. "누가, 어디서, 언제 로그인했는가"라는 정보만 필요하다면 이를 **메트릭**으로 변환하여 저장하는 게 훨씬 경제적입니다.
- **원본 이벤트**: 약 2,500 바이트
- **메트릭 데이터**: 약 150 바이트 (타임스탬프, 사용자, 소스IP, 이벤트코드, 카운트)
- **절감 효과**: 약 94% 라이선스 비용 감소

### 2.2 단계별 실습 가이드

#### 1단계: 비용 유발 요인 파악
어떤 로그가 라이선스를 가장 많이 쓰는지 확인합니다.
```splunk
index=_internal source=*license_usage.log* type="Usage" 
| stats sum(b) as bytes by idx, st 
| eval gb=round(bytes/1024/1024/1024, 2) 
| sort -gb
```

#### 2단계: XML 데이터에서 필드 추출
XML 구조에서 필요한 값만 뽑아냅니다. 예: `<Data Name='TargetUserName'>jsmith</Data>`

**transforms.conf**
```conf
[extract_xml_user]
REGEX = <Data Name='TargetUserName'>(?<user>[^<]+)</Data>
FORMAT = user::$1
WRITE_META = true
```

#### 3단계: mcollect를 이용한 메트릭 생성
스케줄된 검색을 통해 원본 로그를 메트릭 인덱스로 옮깁니다.
```splunk
index=wineventlog EventCode=4624
| bin _time span=1m
| stats count by _time, user, src_ip, dest_host
| rename count as login_count
| mcollect index=wineventlog_metrics name=user_logins
```

#### 4단계: mstats를 이용한 초고속 조회
이제 수백만 건의 원본 로그 대신 메트릭 인덱스를 조회합니다.
```splunk
| mstats sum(login_count) as total_logins where index=wineventlog_metrics AND name=user_logins by user
| sort -total_logins
```
이 방식은 검색 속도가 100배 이상 빠르며 라이선스 사용량도 획기적으로 줄어듭니다.

---

## 3교시: 고급 최적화 전략 및 케이스 스터디 (1시간)

데이터를 줄이는 것만큼 중요한 것이 데이터를 효율적으로 관리하는 것입니다.

### 3.1 데이터 티어링(Data Tiering) 전략

모든 데이터를 비싼 SSD에 보관할 필요는 없습니다.
- **Hot/Warm**: SSD 사용. 실시간 분석용. 7~30일 보관.
- **Cold**: HDD 사용. 과거 데이터 조회용. 30~90일 보관.
- **Frozen**: S3/Glacier 사용. 컴플라이언스용. 1~7년 보관. (검색 시 해동 필요)

### 3.2 요약 인덱스(Summary Indexes) 활용

매일 아침 모든 직원이 보는 대시보드가 10TB의 데이터를 뒤지게 두지 마세요.
- 1시간마다 검색 결과를 요약 인덱스에 저장합니다.
- 대시보드는 요약 인덱스만 바라보게 설정합니다.
- 검색 성능은 올라가고 인덱서 부하는 줄어듭니다.

### 3.3 케이스 스터디: 글로벌 핀테크 기업의 "Project Zero-Waste"

한 글로벌 핀테크 기업은 연간 50억 원의 스플렁크 라이선스 비용을 지불하고 있었습니다.
- **도전 과제**: 하루 150TB의 로그 발생.
- **해결책**:
    1. **Cribl Stream** 도입: 모든 `DEBUG`, `TRACE` 로그를 인덱싱 전 단계에서 삭제 (30TB 절감).
    2. **메트릭 전환**: VPC Flow 로그와 방화벽 로그를 전부 메트릭으로 변환 (60TB 절감).
    3. **직접 전송**: 규정 준수를 위해 보관만 해야 하는 로그 40TB를 스플렁크를 거치지 않고 바로 S3로 전송.
- **결과**: 스플렁크 인덱싱 용량을 하루 20TB로 줄임. 연간 40억 원의 비용 절감 및 보안 분석가들의 검색 속도 대폭 향상.

---

## 자가 진단 퀴즈 (Self-Assessment Quiz)

**문제 1:** 스플렁크에서 특정 데이터를 인덱싱하지 않고 완전히 삭제하기 위해 사용하는 `DEST_KEY`의 값은 무엇인가?
**문제 2:** 데이터 파이프라인 아키텍처에서 스플렁크 앞단에 위치하여 데이터를 필터링하고 S3 등으로 분산 전송하는 데 자주 쓰이는 솔루션은?
**문제 3:** 윈도우 XML 로그처럼 용량이 큰 데이터를 90% 이상 줄여서 저장하기 위해 사용하는 데이터 형식은?
**문제 4:** `mstats` 명령어를 사용하기 위해 데이터를 저장할 때 사용하는 명령어는?

---

## 정답

1. **nullQueue**: 이 값을 지정하면 데이터가 인덱싱 엔진에 도달하기 전에 파기됩니다.
2. **Cribl Stream (또는 Apache Kafka)**: 데이터 파이프라인의 유연성을 높여주는 핵심 도구들입니다.
3. **메트릭 (Metrics)**: 수치형 데이터를 효율적으로 저장하며 검색 속도도 매우 빠릅니다.
4. **mcollect**: 이 명령어를 통해 검색 결과를 메트릭 형식으로 인덱스에 저장할 수 있습니다.

빅테크의 보안은 단순히 해킹을 막는 게 아닙니다. 쏟아지는 데이터 속에서 가성비 좋은 정보를 찾아내는 싸움입니다. 오늘 배운 기술들을 실무에 적용해 보세요. 여러분의 SIEM 운영 능력은 물론, 회사의 비용 절감에도 큰 기여를 할 수 있을 것입니다.
