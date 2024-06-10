---
layout: post
title: "Splunk Week 15: Federated Search & Data Lake Integration"
date: 2024-06-07 09:00:00 +0900
categories: splunk
---

빅테크 보안 운영의 끝판왕은 결국 데이터 스케일링입니다. 로그가 하루에 수십 테라바이트씩 쏟아지는 환경에서 모든 데이터를 Splunk 로컬 스토리지에 담아두는 건 불가능에 가깝습니다. 비용도 문제지만 인덱서 성능이 버티질 못하죠. 오늘은 이 한계를 넘어서는 페더레이티드 서치(Federated Search)와 데이터 레이크 통합 전략을 다룹니다.

### 로컬 스토리지의 한계와 데이터 티어링

Splunk를 처음 운영할 때는 모든 로그를 인덱서에 넣습니다. 하지만 데이터 보존 기간이 길어질수록 스토리지 비용은 기하급수적으로 늘어납니다. 여기서 빅테크 기업들이 선택하는 전략이 바로 데이터 티어링(Data Tiering)입니다.

실시간 분석과 빠른 응답이 필요한 최근 7일에서 30일치 데이터는 Splunk의 Hot/Warm 버킷에 둡니다. 반면 90일이 지난 Cold 데이터나 1년 이상 보관해야 하는 Frozen 데이터는 AWS S3 같은 저렴한 오브젝트 스토리지로 밀어냅니다. S3에 저장된 데이터는 Splunk 인덱서의 자원을 쓰지 않으면서도 필요할 때 꺼내 쓸 수 있는 상태가 됩니다.

### Splunk Federated Search: 경계를 허무는 검색

페더레이티드 서치는 Splunk 외부의 데이터 소스를 마치 내부 인덱스처럼 검색하게 해주는 기술입니다. 예전에는 데이터를 다시 Splunk로 불러와야(Ingest) 했지만, 이제는 데이터가 있는 곳으로 직접 쿼리를 보냅니다.

특히 AWS Athena와의 통합이 핵심입니다. S3에 쌓인 대규모 로그를 Athena가 쿼리하고, 그 결과를 Splunk 서치 헤드에서 바로 보여주는 방식이죠.

#### 설정 워크플로우

1. **AWS 설정**: S3 버킷에 로그를 적재하고 Athena 테이블을 생성합니다.
2. **Splunk Provider 등록**: Settings > Federated Search > Federated Providers에서 AWS Athena를 프로바이더로 등록합니다. IAM Role이나 Access Key가 필요합니다.
3. **Federated Index 생성**: 프로바이더를 선택하고 Athena의 데이터베이스와 테이블을 매핑하는 페더레이티드 인덱스를 만듭니다.

### 실전 SPL 쿼리

설정이 끝나면 검색창에서 바로 외부 데이터를 호출할 수 있습니다. 페더레이티드 인덱스는 `federated:` 접두사를 사용합니다.

```splunk
index=federated:my_s3_data sourcetype=aws:cloudtrail
| stats count by eventName
| sort - count
```

이 쿼리를 실행하면 Splunk는 Athena에 SQL 쿼리를 던지고 결과만 받아옵니다. 로컬 인덱스인 `index=main`과 `union` 명령어로 결합해서 과거 데이터와 현재 데이터를 한 번에 분석할 수도 있습니다.

### 빅테크의 아키텍처: Edge Hub와 Data Fabric Search

글로벌 서비스는 데이터가 전 세계 리전에 흩어져 있습니다. 모든 로그를 한곳으로 모으는 건 네트워크 비용과 지연 시간 때문에 비효율적입니다. 그래서 빅테크는 '데이터 패브릭 서치(Data Fabric Search)' 구조를 씁니다.

각 리전의 Splunk 클러스터를 'Edge Hub'로 두고, 중앙의 서치 헤드가 페더레이티드 서치로 각 허브의 데이터를 훑는 방식입니다. 데이터는 각 지역에 머물되 검색 결과만 중앙으로 모이는 분산형 구조죠. 보안 규정(Data Residency)을 준수하면서도 전사적인 가시성을 확보하는 유일한 방법입니다.

### 셀프 테스트 퀴즈

1. **Splunk 로컬 스토리지 비용을 절감하기 위해 1년 이상 된 로그를 보관하기 가장 적합한 장소는?**
   - 정답: AWS S3 (Frozen/Cold Tier)

2. **페더레이티드 서치에서 외부 데이터 소스를 식별하기 위해 인덱스 이름 앞에 붙이는 접두사는?**
   - 정답: `federated:`

3. **S3에 저장된 로그를 Splunk에서 직접 쿼리하기 위해 중간에서 SQL 엔진 역할을 하는 AWS 서비스는?**
   - 정답: AWS Athena

4. **데이터를 Splunk로 직접 수집하지 않고 원격지에서 검색만 수행할 때의 가장 큰 장점은?**
   - 정답: 인덱싱 비용(License/Storage) 절감 및 데이터 중복 방지

빅테크 보안 전문가라면 툴의 기능에 갇히지 말고 데이터의 흐름과 비용 구조를 설계할 줄 알아야 합니다. 페더레이티드 서치는 그 설계의 핵심 도구입니다.
