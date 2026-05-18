---
layout: post
title: "K8s Deep Week 6: Storage — PV, PVC, StorageClass, CSI Driver"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series storage
---

## 학습 목표

- PV/PVC/StorageClass의 책임 분리를 비유로 안다.
- CSI 드라이버가 클라우드 디스크를 어떻게 K8s 볼륨으로 만드는지 안다.
- ReadWriteOnce/ReadWriteMany/ReadOnlyMany 액세스 모드의 의미와 한계를 안다.
- 동적 프로비저닝 vs 정적 프로비저닝을 구별한다.

## 1. 비유 — "도서관 좌석 예약"

학생(Pod)이 "조용한 자리 30분 쓰고 싶어요" (PVC) 라고 요청합니다.
도서관 시스템(K8s)이 "지정석 7번 비어 있어요, 쓰세요"(PV bind) 라고 답합니다.
도서관 정책(StorageClass)에 따라 "조용한 자리" 가 무엇인지가 정해져 있습니다.
지정석은 도서관 외부 업체(CSI)가 실제 의자·책상을 제공합니다.

K8s 매핑:
- 학생 = Pod
- 예약 요청 = PersistentVolumeClaim (PVC)
- 실제 좌석 = PersistentVolume (PV)
- 도서관 정책 = StorageClass
- 외부 의자 공급자 = CSI Driver

## 2. PV / PVC — 책임 분리

### PV (PersistentVolume)
"이 볼륨이 존재한다. 용량 100Gi, 클러스터 자원." 인프라팀이 정의(정적) 또는 K8s가 자동 생성(동적).

```yaml
apiVersion: v1
kind: PersistentVolume
metadata: { name: pv-100gi }
spec:
  capacity: { storage: 100Gi }
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ssd
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0abc...
```

### PVC (PersistentVolumeClaim)
"나는 100Gi 짜리 ssd 볼륨이 필요해요." 앱 개발자가 정의.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data }
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests: { storage: 100Gi }
  storageClassName: ssd
```

K8s의 PV controller가 매칭되는 PV 찾아 bind. Pod이 PVC를 mount.

## 3. StorageClass — "자동 자판기"

매번 인프라팀이 PV를 손으로 만들면 너무 느리니까, "이 클래스로 요청하면 자동으로 볼륨 만들어 줄게" 같은 자판기.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: ssd }
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

`volumeBindingMode: WaitForFirstConsumer` 는 중요: Pod이 스케줄된 후에 볼륨을 만듦. AZ(가용존)가 맞도록.

## 4. CSI Driver — "표준 외부 인터페이스"

비유: 도서관이 다양한 의자 공급자(LG/한샘/IKEA)와 거래할 때, 매번 다른 API로 주문하면 미칩니다. 그래서 "표준 주문서"를 만들어 모든 공급자가 그 표준을 구현하도록 합니다.

CSI = Container Storage Interface. AWS EBS, GCP PD, Azure Disk, Ceph, Portworx 등 모두 CSI 구현.

### CSI 구조

```
[K8s] ←→ [CSI Controller] ←→ [클라우드 API: 볼륨 생성·삭제·attach]
              ↑
              │
[K8s Pod] ←→ [CSI Node] ←→ [노드에 mount/unmount]
```

- **Controller**: 클러스터에 1~N개. 볼륨 생성·삭제 같은 cluster-scope 작업.
- **Node**: 모든 노드에 DaemonSet. 그 노드에서 mount/format/unmount.

### CSI의 4단계 lifecycle

1. **CreateVolume**: 클라우드에 볼륨 생성 요청.
2. **ControllerPublishVolume (attach)**: 노드에 볼륨 붙임.
3. **NodePublishVolume (mount)**: 노드의 파일시스템에 마운트.
4. (역순으로 unmount/detach/delete)

CSI 깊은 분석은 [보충 3: CSI 깊게] 포스트.

## 5. AccessMode — 함정의 시작

| 모드 | 의미 |
|------|------|
| ReadWriteOnce (RWO) | 한 노드만 read-write |
| ReadWriteMany (RWX) | 여러 노드 동시 read-write |
| ReadOnlyMany (ROX) | 여러 노드 read-only |
| ReadWriteOncePod (RWOP) | 한 Pod만 read-write |

학부생 함정:
- AWS EBS는 RWO만 지원. 여러 노드가 동시 쓰기 불가.
- RWX가 필요하면 EFS/Filestore/Azure Files 같은 file storage 또는 Ceph RBD/CephFS.
- StatefulSet의 각 replica는 자기 PVC를 가짐(RWO로 충분).
- Deployment + RWO PVC + replicas=3 = **2개 Pod이 pending에 빠짐** (한 노드만 mount 가능).

## 6. Volume Snapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: { name: data-snap }
spec:
  volumeSnapshotClassName: ebs-snap
  source: { persistentVolumeClaimName: data }
```

스냅샷 → 새 PVC 복원으로 PIT 백업/복원. Velero가 이걸 활용 (DR 강의에서).

## 7. StatefulSet과 Volume

StatefulSet은 Pod마다 안정적 이름(`pod-0`, `pod-1`)과 자기 PVC를 가집니다. `volumeClaimTemplates` 로 PVC 자동 생성.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: kafka }
spec:
  serviceName: kafka
  replicas: 3
  template: { ... }
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        resources: { requests: { storage: 100Gi } }
        storageClassName: ssd
```

Pod 0/1/2 각자 100Gi PVC. Pod 재시작해도 같은 PVC mount.

## 8. 운영 함정 5선

1. **Retain reclaimPolicy 안 둠**: PVC 삭제 시 PV·데이터도 사라짐. 운영은 Retain 권장.
2. **volumeBindingMode Immediate**: AZ 불일치로 scheduling 실패.
3. **RWX 필요한데 EBS 쓰기**: 늘 Pending.
4. **CSI 버전 미스매치**: 드라이버와 K8s 버전 호환표 확인.
5. **백업 없음**: snapshot 없으면 데이터 삭제 시 회복 불가.

## 9. 실습

```bash
# 1. AWS EBS CSI driver 설치 (EKS 환경에서)
# 2. StorageClass 만들기 (gp3, encrypted)
# 3. PVC + Pod 만들어 mount
kubectl exec -it pvc-pod -- sh -c 'echo hi > /data/test.txt'
# 4. Pod 삭제 후 같은 PVC mount하는 새 Pod → test.txt 잔존 확인
# 5. VolumeSnapshot 으로 PIT 백업
# 6. snapshot에서 새 PVC 복원
```

## 10. 자가평가 퀴즈

### Q1. PV와 PVC의 책임 분리?
1. **PV는 인프라 측, PVC는 앱 측 요청**
2. 둘이 같은 것
3. PVC가 인프라
4. 무관

**정답: 1.**

### Q2. WaitForFirstConsumer를 쓰는 이유?
1. **Pod 스케줄 후 볼륨 만들기 → AZ 일치 보장**
2. 더 빠름
3. UI
4. 무관

**정답: 1.**

### Q3. AWS EBS가 RWX 안 되는 이유?
1. **블록 디바이스는 본질적으로 한 호스트만 mount 가능**
2. AWS 정책
3. K8s 한계
4. 무관

**정답: 1.**

### Q4. StatefulSet의 volumeClaimTemplates 효과?
1. **각 Pod이 자기 PVC를 자동 가짐 (재시작해도 같은 볼륨)**
2. 모두 같은 볼륨
3. UI
4. 무관

**정답: 1.**

### Q5. Retain reclaimPolicy의 가치?
1. **PVC 삭제 시 PV·데이터 보존 — 실수 방지**
2. 더 빠름
3. 자동 삭제
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 7: Scheduler 깊게]에서 점수 알고리즘과 커스텀 scheduler 작성을 다룹니다.
