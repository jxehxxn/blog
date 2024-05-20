---
layout: post
title: "[Splunk 11주차] 대규모 클러스터 관리와 아키텍처: 무너지지 않는 데이터 엔진 만들기"
date: 2024-05-10 09:00:00 +0900
categories: splunk
---

빅테크 기업에서 보안 전문가로 일하며 수많은 장애를 겪었습니다. 그중 가장 끔찍한 기억은 데이터가 쏟아지는데 검색 엔진이 멈춰버리는 순간입니다. 스플렁크(Splunk)를 단순히 설치하는 것과 대규모 환경에서 운영하는 것은 완전히 다른 이야기입니다. 11주차 수업에서는 시스템이 스스로를 지탱하게 만드는 클러스터링과 아키텍처를 다룹니다.

### 인덱서 클러스터링 (Indexer Clustering)

데이터를 저장하는 인덱서가 하나라도 죽으면 보안 관제는 마비됩니다. 이를 방지하기 위해 인덱서 클러스터링을 사용합니다. 핵심은 복제 계수(Replication Factor, RF)와 검색 계수(Search Factor, SF)입니다. RF는 데이터 복사본의 개수를 정하고, SF는 즉시 검색 가능한 데이터의 개수를 결정합니다.

설정은 `server.conf` 파일에서 관리합니다. 클러스터 매니저(Cluster Manager) 노드에서 아래와 같은 설정을 확인해야 합니다.

```ini
[clustering]
mode = manager
replication_factor = 3
search_factor = 2
pass4SymmKey = your_secret_key
```

피어(Peer) 노드들은 매니저의 지시를 받아 데이터를 복제합니다. 한 대가 고장 나도 다른 노드가 즉시 데이터를 채워 넣습니다.

### 서치 헤드 클러스터링 (Search Head Clustering, SHC)

사용자가 검색을 수행하는 서치 헤드도 묶어야 합니다. 여러 명의 분석가가 동시에 무거운 쿼리를 던지면 단일 서버는 버티지 못합니다. SHC는 캡틴(Captain)이라는 리더 노드가 작업을 분배합니다.

설정 변경은 개별 서버가 아니라 디플로이어(Deployer)를 통해 배포합니다. `shclustering` 섹션에서 클러스터 멤버십을 정의합니다.

```ini
[shclustering]
conf_deploy_fetch_url = https://deployer:8089
mgmt_uri = https://sh-node1:8089
pass4SymmKey = your_secret_key
```

### 데이터 생애주기 (Data Lifecycle)

데이터는 시간이 지나면 가치가 변합니다. 모든 데이터를 비싼 SSD에 영원히 둘 수는 없습니다. 스플렁크는 데이터를 네 단계로 나눕니다.

1. **Hot**: 현재 쓰여지고 있는 데이터입니다. 가장 빠릅니다.
2. **Warm**: 쓰기는 끝났지만 자주 검색되는 데이터입니다.
3. **Cold**: 오래된 데이터로, 저렴한 HDD로 옮겨집니다.
4. **Frozen**: 보관 기간이 지나 삭제되거나 아카이브로 이동합니다.

`indexes.conf` 파일에서 이 주기를 제어합니다.

```ini
[main]
homePath = $SPLUNK_DB/main/db
coldPath = $SPLUNK_DB/main/colddb
thawedPath = $SPLUNK_DB/main/thaweddb
maxDataSize = auto_high_volume
maxHotBuckets = 10
frozenTimePeriodInSecs = 7776000
```

`frozenTimePeriodInSecs` 설정이 90일(7,776,000초)을 의미합니다. 이 시간이 지나면 데이터는 자동으로 사라집니다.

### 역할 기반 접근 제어 (RBAC)

보안 전문가로서 가장 신경 쓰는 부분입니다. 모든 사용자가 모든 인덱스를 볼 수 있어서는 안 됩니다. 역할(Role)을 만들고 권한(Capability)을 부여하세요. 특정 인덱스만 검색할 수 있도록 제한하는 것이 기본입니다.

빅테크 환경에서는 보통 팀별로 인덱스를 나눕니다. 보안팀은 `index=security`만 보고, 개발팀은 `index=app_log`만 보게 설정합니다.

### 빅테크의 인덱스 설계 전략

대규모 환경에서는 인덱스 크기 산정이 생명입니다. 하루 유입량(Daily Ingest)을 계산하고 1.5배 정도의 여유 공간을 확보하세요. 보관 기간(Retention)은 법적 규제와 비용을 고려해 결정합니다. 보통 실시간 분석용은 30일, 감사용은 1년 이상으로 설정합니다.

---

### 자가 진단 퀴즈 (Self-Assessment Quiz)

**문제 1: 인덱서 클러스터에서 데이터 복제본의 개수를 결정하는 설정값은 무엇인가요?**
- 정답: Replication Factor (RF)

**문제 2: 서치 헤드 클러스터에서 설정을 배포하는 역할을 하는 노드의 명칭은?**
- 정답: 디플로이어 (Deployer)

**문제 3: `indexes.conf`에서 데이터가 삭제되기 전까지 보관되는 시간을 초 단위로 설정하는 옵션은?**
- 정답: `frozenTimePeriodInSecs`

**문제 4: 데이터 생애주기 중 쓰기가 진행 중인 상태의 버킷 이름은?**
- 정답: Hot Bucket

**문제 5: 클러스터 매니저와 피어 노드 간의 통신 보안을 위해 사용하는 키 설정 이름은?**
- 정답: `pass4SymmKey`

클러스터링은 복잡하지만 시스템의 안정성을 보장하는 유일한 방법입니다. 다음 주에는 성능 최적화에 대해 알아보겠습니다.
