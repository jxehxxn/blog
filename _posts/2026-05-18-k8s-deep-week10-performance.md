---
layout: post
title: "K8s Deep Week 10: Performance & Scale — etcd 튜닝, API Priority & Fairness, 대규모 클러스터"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series performance
---

## 학습 목표

- 클러스터 크기에 따른 병목 위치를 안다.
- etcd 튜닝 핵심 파라미터를 이해한다.
- API Priority and Fairness(APF) 모델을 안다.
- 5,000 노드 클러스터 운영 사례의 핵심 결정을 학습한다.

## 1. 비유 — "10명 식당 vs 1만명 결혼식"

10명 식당은 직원 1명이면 충분. 1만명 결혼식은 (1) 입장 통제 (2) 자리 배정 (3) 음식 줄 관리 (4) 청소가 별도 팀이 되어야 합니다. K8s도 작은 클러스터에서는 기본값으로 충분하지만, 5,000 노드면 모든 컴포넌트가 튜닝 대상.

## 2. 클러스터 크기별 병목

| 크기 | 첫 병목 | 두 번째 |
|------|---------|---------|
| ~50 노드 | 없음 (기본값 OK) | — |
| 50~500 | scheduler 처리량 | controller-manager |
| 500~2000 | API server CPU | etcd I/O |
| 2000~5000 | etcd disk latency | network bandwidth |
| 5000+ | etcd watch fanout | API server memory |

## 3. etcd 튜닝

### Disk
- **NVMe SSD 필수**. HDD 절대 금지.
- fsync latency p99 < 10ms 목표 (`etcd_disk_wal_fsync_duration_seconds`).
- 전용 디스크, 다른 워크로드와 분리.

### 메모리
- 권장: 2000 노드 클러스터에 etcd 메모리 16 GB.
- etcd DB 크기 한계: 기본 2 GB, `--quota-backend-bytes`로 8 GB 까지.

### Defragmentation
etcd는 MVCC라 시간이 지나면 디스크가 부풉니다. 주기적 defrag.

```bash
etcdctl --endpoints=https://node:2379 defrag
```

빅테크 권장: 주 1회 cron (rolling으로 한 노드씩, leader 마지막).

### 네트워크
- etcd 노드 간 RTT 10ms 이하.
- 같은 region, 가능하면 같은 AZ (단 AZ 장애로 인한 quorum loss 주의 → AZ 분산도 균형).

## 4. API Priority and Fairness (APF)

비유: 식당이 바쁠 때 VIP / 단골 / 일반 손님을 따로 줄 세움. 한 손님이 무한 주문해도 다른 줄은 영향 없음.

기존: API server에 동시 요청이 몰리면 모든 요청이 같은 큐 → 한 사용자가 폭주하면 전체 장애.

APF는 FlowSchema + PriorityLevelConfiguration 으로 요청을 카테고리화:

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata: { name: my-controller }
spec:
  priorityLevelConfiguration: { name: workload-high }
  matchingPrecedence: 100
  distinguisherMethod: { type: ByUser }
  rules:
    - subjects:
        - kind: ServiceAccount
          serviceAccount: { name: my-operator, namespace: operators }
      resourceRules:
        - verbs: ["*"]
          apiGroups: ["*"]
          resources: ["*"]
```

기본 PriorityLevel: system, leader-election, workload-high, workload-low, global-default, exempt, catch-all.

빅테크에서 자체 controller마다 적절한 priority 지정 → API server 과부하에서도 critical path 보호.

## 5. Scheduler 성능

5000 노드면 매 Pod scheduling에 모든 노드 평가 = 5000번 점수 계산.

최적화:
- `percentageOfNodesToScore`: 기본 50% (= 2500 노드만 평가하고 끝). 큰 클러스터에서 10~30%로 낮춤.
- Score plugin 줄이기: 불필요한 plugin 비활성.
- Custom plugin: 도메인 특화 빠른 점수.

## 6. Controller 부하

각 controller가 자체 informer로 watch. namespace 수십만 + Pod 수십만이면 controller-manager 메모리 폭증.

대책:
- informer cache TTL 조절.
- 일부 controller를 별도 process로 분리.
- HPA shard / per-namespace ownership.

## 7. CoreDNS 성능

DNS 쿼리 폭주가 잦은 함정. Pod이 매 connection마다 DNS lookup → CoreDNS 과부하.

대책:
- NodeLocal DNSCache: 각 노드에 캐시 데몬.
- `ndots: 1` 또는 `ndots: 2` (기본 5는 너무 많은 lookup 발생).
- Pod DNS policy 튜닝.

## 8. Ingress / Service 규모

iptables 규칙은 Service 수에 O(n)로 증가. 5,000 Service면 iptables 평가 시간이 ms 단위.

대책:
- kube-proxy IPVS 모드.
- 또는 Cilium eBPF (kube-proxy 대체).

## 9. 빅테크 사례 — Google Borg 영감의 5000 노드 운영

- etcd: 5 노드 클러스터, NVMe, 별도 디스크, 주간 defrag.
- API server: 5 replica, APF로 SA별 priority.
- scheduler: 2 replica + percentageOfNodesToScore 10%.
- controller-manager: 메모리 64 GB, leader election 단일.
- 100,000 Pod까지 안정 운영 보고.

## 10. 관측 — 무엇을 봐야 하나

```promql
# API server latency
histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (le, verb))

# etcd I/O
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))

# scheduler latency
histogram_quantile(0.99, rate(scheduler_e2e_scheduling_duration_seconds_bucket[5m]))

# work queue depth (controller)
workqueue_depth
```

p99 > 임계 → 알람.

## 11. 운영 함정 5선

1. etcd HDD → 모든 클러스터 작업 lag.
2. APF 미설정 → 한 사용자가 클러스터 마비.
3. CoreDNS 미튜닝 → DNS 폭주.
4. iptables 모드 + 5000 Service → 네트워크 느림.
5. scheduler 단일 replica → 장애 시 신규 Pod 전부 pending.

## 12. 실습

```bash
# 1. etcd fsync metric 확인
kubectl get --raw /metrics | grep etcd_disk_wal_fsync_duration

# 2. API server APF state
kubectl get --raw /debug/api_priority_and_fairness/dump_priority_levels

# 3. CoreDNS 메트릭
kubectl -n kube-system port-forward svc/kube-dns 9153:9153
curl localhost:9153/metrics | grep coredns_dns_request_duration_seconds

# 4. NodeLocal DNSCache 배포
```

## 13. 자가평가 퀴즈

### Q1. 1000 노드 클러스터의 가장 흔한 첫 병목?
1. **API server CPU 또는 etcd I/O**
2. 디스크 용량
3. 메모리
4. 무관

**정답: 1.**

### Q2. APF의 핵심 가치?
1. **요청을 카테고리화해 한 사용자 폭주가 전체에 영향 안 가게**
2. UI 개선
3. 비용
4. 무관

**정답: 1.**

### Q3. percentageOfNodesToScore 낮추는 효과?
1. **scheduling 빨라짐 / 약간 덜 최적**
2. 더 정확
3. 무관
4. 비용

**정답: 1.**

### Q4. etcd defrag 안 하면?
1. **DB 디스크 크기가 부풀어 성능 저하 + quota 초과**
2. 안전
3. 무관
4. 비용

**정답: 1.**

### Q5. CoreDNS 폭주 흔한 원인?
1. **Pod ndots: 5 기본값으로 lookup 폭증**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 14. 다음 주차

[Week 11: Multi-tenancy]에서 namespace, hierarchical namespace, vCluster 같은 격리 패턴을 다룹니다.
