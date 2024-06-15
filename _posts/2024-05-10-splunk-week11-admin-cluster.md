---
layout: post
title: "[Splunk 11주차] 대규모 클러스터 관리와 아키텍처: 무너지지 않는 데이터 엔진 만들기"
date: 2024-05-10 09:00:00 +0900
categories: splunk
---

# 제 11장: 스플렁크 클러스터 아키텍처와 대규모 데이터 관리

빅테크 기업에서 보안 전문가로 일하며 수많은 장애를 겪었습니다. 그중 가장 끔찍한 기억은 데이터가 쏟아지는데 검색 엔진이 멈춰버리는 순간입니다. 스플렁크(Splunk)를 단순히 설치하는 것과 대규모 환경에서 운영하는 것은 완전히 다른 이야기입니다. 11주차 수업에서는 시스템이 스스로를 지탱하게 만드는 클러스터링과 아키텍처를 다룹니다.

이 장에서는 10대 이상의 인덱서로 구성된 대규모 클러스터를 설계하고 운영하는 방법, 그리고 데이터가 생성되어 소멸하기까지의 전 과정을 심도 있게 분석합니다.

---

## 1. 인덱서 클러스터링 (Indexer Clustering) 심층 분석

인덱서 클러스터링은 데이터의 가용성(Availability)과 복구 능력(Resiliency)을 보장하기 위한 핵심 기술입니다. 단순히 여러 대의 서버를 묶는 것을 넘어, 데이터의 복제와 검색 가능 상태를 지능적으로 관리합니다.

### 1.1 클러스터 구성 요소 (Cluster Components)

1.  **클러스터 매니저 (Cluster Manager, CM)**:
    *   클러스터의 '두뇌' 역할을 수행합니다.
    *   모든 피어 노드의 상태를 모니터링하고, 어떤 노드가 어떤 버킷(Bucket)을 가지고 있는지 관리합니다.
    *   복제 계수(RF)와 검색 계수(SF)가 충족되지 않을 경우, 피어 노드에게 복제 작업을 지시합니다.
    *   검색 헤드가 쿼리를 던질 때, 어떤 피어 노드에서 데이터를 가져와야 하는지 알려주는 '버킷 맵'을 제공합니다.

2.  **피어 노드 (Peer Nodes)**:
    *   실제 데이터를 저장하고 검색을 수행하는 '일꾼'입니다.
    *   데이터가 유입되면 이를 인덱싱하고, 설정된 RF/SF에 따라 다른 피어 노드로 데이터를 복제합니다.
    *   매니저 노드와 주기적으로 하트비트(Heartbeat)를 주고받으며 자신의 상태를 보고합니다.

3.  **서치 헤드 (Search Heads)**:
    *   사용자의 쿼리를 처리합니다.
    *   매니저 노드로부터 검색에 필요한 버킷 위치 정보를 받아온 뒤, 해당 피어 노드들에게 병렬로 검색을 요청합니다.

### 1.2 복제 계수(RF)와 검색 계수(SF)의 메커니즘

*   **Replication Factor (RF)**: 클러스터 전체에 유지해야 하는 데이터 복사본의 총 개수입니다. RF=3이라면, 원본 데이터를 포함해 총 3개의 복사본이 서로 다른 피어 노드에 저장됩니다. 이는 하드웨어 장애 시 데이터 유실을 방지합니다.
*   **Search Factor (SF)**: 즉시 검색 가능한 상태(Searchable)로 유지해야 하는 데이터 복사본의 개수입니다. SF는 항상 RF보다 작거나 같아야 합니다. SF=2라면, 3개의 복사본 중 2개는 인덱스 파일(TSIDX)을 포함하여 즉시 검색이 가능한 상태로 유지됩니다. 나머지 1개는 원본 데이터(Rawdata)만 가지고 있다가, 필요할 때 인덱싱을 수행합니다.

---

## 2. 데이터 버킷의 생애주기와 '버킷 롤(Bucket Roll)'

스플렁크는 데이터를 '버킷(Bucket)'이라는 단위로 관리합니다. 버킷은 시간에 따라 상태가 변하며, 이를 '데이터 생애주기'라고 부릅니다.

### 2.1 버킷의 4단계 상태

1.  **Hot (쓰기 가능)**:
    *   현재 데이터가 유입되어 기록되고 있는 상태입니다.
    *   메모리와 디스크를 동시에 사용하며, 가장 빠른 접근 속도를 가집니다.
    *   인덱서당 여러 개의 Hot 버킷이 존재할 수 있습니다.

2.  **Warm (읽기 전용)**:
    *   Hot 버킷이 가득 차거나 특정 조건에 의해 닫힌 상태입니다.
    *   데이터는 여전히 고성능 저장소(SSD)에 머물며 빈번한 검색에 대응합니다.

3.  **Cold (장기 보관)**:
    *   시간이 지나 검색 빈도가 낮아진 데이터입니다.
    *   비용 절감을 위해 저렴한 저장소(HDD)로 이동됩니다.
    *   검색 속도는 Warm에 비해 느리지만, 여전히 검색 가능합니다.

4.  **Frozen (삭제 또는 아카이브)**:
    *   설정된 보관 기간이 지난 데이터입니다.
    *   기본적으로 삭제되지만, 설정에 따라 외부 저장소(S3, Tape 등)로 아카이브할 수 있습니다.

### 2.2 버킷 롤(Bucket Roll: Hot -> Warm)의 내부 동작

Hot 버킷이 Warm으로 전환되는 과정을 '버킷 롤'이라고 합니다. 이 과정에서 발생하는 일은 다음과 같습니다.

1.  **트리거 발생**: `maxDataSize` (크기), `maxHotBuckets` (개수), `maxHotSpanSecs` (시간) 중 하나라도 충족되면 롤이 시작됩니다.
2.  **파일 닫기**: 현재 기록 중인 저널 파일과 인덱스 파일을 닫습니다.
3.  **디렉토리 이름 변경**: `hot_v1_0`과 같은 임시 이름에서 `db_1712345678_1712345000_0` (최신시간_최초시간_ID) 형식으로 이름을 변경합니다.
4.  **매니저 보고**: 피어 노드는 클러스터 매니저에게 "새로운 Warm 버킷이 생성되었다"고 보고합니다.
5.  **복제 트리거**: 매니저는 해당 버킷의 RF/SF를 확인하고, 부족할 경우 다른 피어 노드에게 복제를 지시합니다. 이때 Rawdata와 TSIDX 파일이 네트워크를 통해 전송됩니다.

---

## 3. 10노드 인덱서 클러스터 실전 구성 (Configuration)

대규모 환경에서는 설정 파일 관리가 핵심입니다. 10대의 인덱서를 효율적으로 운영하기 위한 `server.conf`와 `indexes.conf` 예시입니다.

### 3.1 Cluster Manager 설정 (`server.conf`)

매니저 노드에서는 클러스터의 정책을 정의합니다.

```ini
[general]
serverName = splunk-cm-01
pass4SymmKey = massive_cluster_secret_2024

[clustering]
mode = manager
# 10노드 환경에서는 가용성을 위해 RF=3, SF=2 권장
replication_factor = 3
search_factor = 2
# 피어 노드 간 데이터 전송 시 사용할 포트
cluster_label = production_cluster

[indexer_discovery]
# 피어 노드들이 매니저를 자동으로 찾을 수 있게 설정
pass4SymmKey = discovery_secret_key
```

### 3.2 Peer Node 설정 (`server.conf`)

각 피어 노드는 매니저의 위치를 알고 있어야 합니다.

```ini
[general]
serverName = splunk-idx-01  # 각 노드마다 고유해야 함
pass4SymmKey = massive_cluster_secret_2024

[replication_port://9887]
# 데이터 복제를 위한 전용 포트

[clustering]
mode = peer
manager_uri = https://10.0.0.10:8089
pass4SymmKey = massive_cluster_secret_2024

[indexer_discovery]
# 인덱서 디스커버리 활성화 (포워더 연결용)
master_uri = https://10.0.0.10:8089
pass4SymmKey = discovery_secret_key
```

### 3.3 대규모 환경을 위한 `indexes.conf` (Volume 관리)

모든 인덱서에 동일하게 적용되어야 하므로, 매니저의 `manager-apps` 디렉토리에서 관리합니다.

```ini
# 볼륨 설정을 통해 저장소 공간을 효율적으로 관리
[volume:primary]
path = /opt/splunk/var/lib/splunk
# SSD 공간 제한 (예: 2TB)
maxVolumeDataSizeMB = 2000000

[volume:cold_storage]
path = /mnt/splunk_cold_storage
# HDD 공간 제한 (예: 10TB)
maxVolumeDataSizeMB = 10000000

# 기본 인덱스 설정
[default]
# 클러스터링 활성화
repFactor = auto

[security_logs]
homePath = volume:primary/security_logs/db
coldPath = volume:cold_storage/security_logs/colddb
thawedPath = $SPLUNK_DB/security_logs/thaweddb
# Hot 버킷 크기 제어 (대규모 환경에서는 크게 설정)
maxDataSize = auto_high_volume
# 최대 Hot 버킷 개수
maxHotBuckets = 10
# 보관 기간: 90일 (7,776,000초)
frozenTimePeriodInSecs = 7776000
# 인덱스별 용량 제한 (예: 5TB)
maxTotalDataSizeMB = 5000000

[app_logs]
homePath = volume:primary/app_logs/db
coldPath = volume:cold_storage/app_logs/colddb
thawedPath = $SPLUNK_DB/app_logs/thaweddb
# 보관 기간: 30일
frozenTimePeriodInSecs = 2592000
maxTotalDataSizeMB = 2000000
```

---

## 4. 서치 헤드 클러스터링 (Search Head Clustering, SHC)

인덱서가 데이터를 저장한다면, 서치 헤드는 사용자에게 결과를 보여줍니다. 대규모 환경에서는 서치 헤드 역시 클러스터로 구성해야 합니다.

### 4.1 SHC의 핵심 개념

*   **캡틴 (Captain)**: 클러스터의 리더입니다. 검색 작업을 스케줄링하고, 지식 객체(Knowledge Objects)의 복제를 관리합니다. 캡틴은 멤버들 간의 투표(Raft 알고리즘)를 통해 선출됩니다.
*   **디플로이어 (Deployer)**: SHC 멤버들에게 앱과 설정을 배포하는 독립적인 서버입니다.
*   **검색 아티팩트 복제**: 한 서치 헤드에서 수행한 검색 결과(Artifact)는 다른 멤버들에게 복제되어, 사용자가 어떤 노드에 접속하더라도 동일한 결과를 빠르게 볼 수 있게 합니다.

### 4.2 SHC 설정 예시 (`server.conf`)

```ini
[shclustering]
conf_deploy_fetch_url = https://10.0.0.50:8089
mgmt_uri = https://10.0.0.21:8089
pass4SymmKey = shc_secret_key
shcluster_label = production_shc
```

---

## 5. 운영 베스트 프랙티스 (Operational Best Practices)

1.  **유지보수 모드 (Maintenance Mode)**: 클러스터 매니저나 피어 노드를 재시작할 때는 반드시 유지보수 모드를 활성화하세요. 그렇지 않으면 매니저가 노드 다운을 장애로 판단하고 불필요한 데이터 복제를 시작하여 네트워크 부하를 초래합니다.
    *   명령어: `splunk enable maintenance-mode`
2.  **사이트 인식 클러스터링 (Multisite Clustering)**: 데이터 센터가 두 곳 이상이라면 Multisite 설정을 통해 재해 복구(DR) 환경을 구축하세요. 각 사이트에 데이터 복사본을 강제로 분산 저장할 수 있습니다.
3.  **모니터링 콘솔 (Monitoring Console, MC)**: 클러스터의 건강 상태, RF/SF 충족 여부, 버킷 롤 현황을 실시간으로 확인하세요. MC는 스플렁크 관리자의 가장 강력한 무기입니다.

---

## 6. 자가 진단 퀴즈 (Advanced Quiz)

**문제 1: RF=3, SF=2인 클러스터에서 피어 노드 2대가 동시에 장애가 발생했을 때, 데이터 유실 여부와 검색 가능 여부를 설명하시오.**
- 정답: RF=3이므로 데이터 복사본이 1개 남아있어 데이터 유실은 발생하지 않습니다. 하지만 SF=2인데 2대가 죽었으므로 검색 가능한 복사본(TSIDX)이 남아있지 않을 수 있습니다. 이 경우 남은 1개의 복사본을 기반으로 인덱싱을 다시 수행할 때까지 해당 데이터는 검색이 불가능할 수 있습니다.

**문제 2: Hot 버킷이 Warm으로 전환되는 '버킷 롤'의 트리거 조건 3가지를 쓰시오.**
- 정답: `maxDataSize` (버킷 크기 초과), `maxHotBuckets` (최대 Hot 버킷 개수 초과), `maxHotSpanSecs` (버킷의 시간 범위 초과).

**문제 3: 클러스터 매니저를 재시작하기 전, 데이터 재복제(Re-fix)를 방지하기 위해 실행해야 하는 설정은?**
- 정답: 유지보수 모드 (Maintenance Mode) 활성화.

**문제 4: `indexes.conf`에서 SSD와 HDD를 나누어 관리하기 위해 사용하는 설정 섹션의 이름은?**
- 정답: `[volume:<name>]`

**문제 5: 서치 헤드 클러스터에서 설정을 배포하고 관리하는 서버의 명칭은?**
- 정답: 디플로이어 (Deployer)

---

클러스터링은 복잡하지만 시스템의 안정성을 보장하는 유일한 방법입니다. 이 텍스트북의 내용을 숙지한다면, 수만 대의 서버에서 발생하는 로그를 안정적으로 처리하는 인프라를 구축할 수 있을 것입니다. 다음 장에서는 성능 최적화와 트러블슈팅에 대해 깊이 있게 다루겠습니다.
