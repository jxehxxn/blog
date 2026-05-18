---
layout: post
title: "K8s Deep 보충 4: etcd 운영 — Backup, Restore, Defrag, 멀티노드"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series supplement etcd
---

10주차에서 짧게 다룬 etcd 운영을 깊게 정리합니다.

## 1. 비유 — "캠퍼스 학적 데이터베이스"

학적 데이터가 사라지면 학생 정보 전체 손실. 그래서 (1) 매일 백업, (2) 정기 정리(defrag), (3) 다중 사본(3개 노드), (4) 복구 절차 연습 (DR drill).

etcd가 정확히 같은 운영을 요구합니다.

## 2. Backup — 한 줄로 시작

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://node:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-$(date +%Y%m%d-%H%M).db
```

snapshot은 consistent point-in-time. 매시간 cron + S3 적재 표준.

## 3. Restore

```bash
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-20260518-1400.db \
  --data-dir=/var/lib/etcd-restored
```

restore는 새 data-dir로. 다음 단계: API server를 멈추고 etcd가 새 data-dir 가리키도록 manifest 수정 후 재시작.

빅테크 표준: 분기 1회 DR drill로 실제 restore 절차 검증. "백업이 있는데 복원이 안 되는" 사고 흔함.

## 4. Defragmentation

etcd는 MVCC. 오래된 revision이 누적되어 DB 파일 커짐. 일정 임계(예: DB 크기의 30% 가비지) 이상이면 defrag로 압축.

```bash
etcdctl --endpoints=https://node:2379 defrag
```

권장:
- 자동: `--auto-compaction-mode=periodic --auto-compaction-retention=8h` 옵션.
- 수동 defrag: 주 1회, rolling (한 노드씩, leader는 마지막).

주의: defrag 중 노드는 read 가능, write 일시 차단. 작은 클러스터 (3 노드)에서 두 노드 동시 defrag는 quorum loss 위험.

## 5. Multi-node etcd

### 권장 노드 수
- 1: 학습용. 장애 시 cluster 정지.
- 3: 표준. 1 노드 장애 허용.
- 5: 큰 클러스터. 2 노드 장애 허용.
- 짝수(2/4)는 절대 금지 — quorum 계산 불리.

### Quorum 공식
N 노드 클러스터의 quorum = ⌊N/2⌋ + 1.
- 3 노드: 2개 필요.
- 5 노드: 3개 필요.

### Leader 선출
follower들이 leader 응답 없으면 election timeout 후 새 leader 선출. 평소 200ms~1초.

## 6. 모니터링 — 무엇을 봐야 하나

```promql
# fsync latency (가장 중요)
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))
# < 10ms 목표

# leader change (잦으면 문제)
sum(increase(etcd_server_leader_changes_seen_total[1h]))

# DB 크기
etcd_mvcc_db_total_size_in_bytes

# quota
etcd_server_quota_backend_bytes

# proposal commit duration
histogram_quantile(0.99, rate(etcd_server_proposals_commit_duration_seconds_bucket[5m]))
```

알람:
- fsync p99 > 20ms.
- leader change > 5/hour.
- DB 크기 > 80% quota.

## 7. 분리 운영

`--initial-cluster=` 로 다른 노드와의 통신 설정. K8s가 etcd를 static pod로 띄울 때 kubeadm이 자동 구성. 외부 etcd cluster를 쓰면 API server `--etcd-servers=` 옵션으로 연결.

빅테크 패턴: 컨트롤 플레인 노드와 etcd 노드를 분리 (전용 etcd 노드). 자원 경합 차단.

## 8. Encryption at Rest

9주차에서 다룸. 모든 secret은 AES 또는 KMS provider로 etcd 안에서 암호화.

## 9. 함정 모음

1. **HDD 사용**: fsync 지연으로 cluster lag.
2. **짝수 노드**: quorum 계산 불리.
3. **defrag 동시 실행**: quorum loss.
4. **DB quota 도달**: 모든 write 거부. 알람 필수.
5. **백업 검증 없음**: 백업 파일이 손상돼도 모름.

## 10. 학부 실습

```bash
# 1. kind 클러스터에서 etcd 백업
docker exec -it kind-control-plane etcdctl ...

# 2. 의도적으로 namespace 삭제
kubectl delete ns default

# 3. snapshot restore → 복원

# 4. 메트릭 보기
kubectl get --raw /metrics | grep etcd_disk_wal_fsync
```

## 11. 결론

etcd가 K8s의 진실의 원천. backup/restore/defrag/monitoring 4종 운영을 손에 익혀야 시니어. 모든 사고 분석의 출발점이기도 합니다.
