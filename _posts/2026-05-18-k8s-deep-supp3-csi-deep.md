---
layout: post
title: "K8s Deep 보충 3: CSI 깊게 — Block, File, Object Storage 그리고 드라이버 내부"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series supplement storage
---

6주차에서 다룬 CSI를 드라이버 내부 + 스토리지 종류별로 풉니다.

## 1. 3가지 storage 종류

### Block Storage (EBS/PD)
- 비유: USB 외장 디스크. 한 컴퓨터에 꽂으면 그 컴퓨터만 씀.
- ReadWriteOnce.
- 빠름, 낮은 latency.
- DB 워크로드 표준.

### File Storage (EFS, Filestore, Azure Files, CephFS)
- 비유: 사내 공유 폴더. 여러 컴퓨터가 동시 쓰기 가능.
- ReadWriteMany 가능.
- Block보다 느림.
- 공유 콘텐츠, 로그 적재.

### Object Storage (S3, GCS, Azure Blob)
- 비유: 사진 클라우드. URL로 access.
- POSIX 파일시스템 아님 (random write 어려움).
- CSI 직접보다 application code에서 SDK로 접근이 일반적.
- 단, s3fs, goofys 같은 FUSE 마운트 CSI 존재.

## 2. CSI 드라이버 아키텍처 깊게

```
[K8s API server]
    ↓ provision 요청
[external-provisioner sidecar] ── gRPC ──→ [CSI controller plugin]
                                              ↓
                                          [클라우드 API: CreateVolume]

[Pod schedule]
    ↓
[external-attacher sidecar] ── gRPC ──→ [CSI controller plugin]
                                          ↓
                                      [클라우드 API: AttachVolume]

[Pod 시작]
    ↓
[kubelet] ── CSI socket ──→ [CSI node plugin (DaemonSet)]
                              ↓
                          [Linux mount syscall]
```

핵심: external sidecar (provisioner, attacher, snapshotter, resizer)가 K8s 측 로직을 수행하고, CSI 드라이버 자체는 표준 gRPC만 구현.

## 3. CSI 명세 — 주요 RPC

### Controller Service
- CreateVolume / DeleteVolume
- ControllerPublishVolume (attach) / ControllerUnpublishVolume (detach)
- CreateSnapshot / DeleteSnapshot
- ControllerExpandVolume (volume 크기 변경)

### Node Service
- NodeStageVolume (글로벌 mount)
- NodePublishVolume (Pod별 mount)
- NodeUnpublishVolume / NodeUnstageVolume
- NodeGetCapabilities

이 표준만 구현하면 K8s가 그 스토리지를 자동 인식.

## 4. AWS EBS CSI 한 줄씩

```yaml
# StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: gp3 }
provisioner: ebs.csi.aws.com
parameters:
  type: gp3              # gp2/gp3/io1/io2/st1/sc1
  encrypted: "true"
  kmsKeyId: arn:aws:kms:...:key/...
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

`allowVolumeExpansion: true` 가 있어야 PVC를 늘렸을 때 자동 확장.

## 5. EFS CSI

ReadWriteMany 필요할 때. dynamic provisioning은 EFS Access Point 사용.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: efs-shared }
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-...
  directoryPerms: "0755"
```

여러 노드의 여러 Pod이 동시 read-write.

## 6. CephFS / RBD CSI (셀프호스트)

OnPrem 또는 멀티클라우드의 표준 선택지: Rook이 Ceph을 K8s로 패키징.

- CephFS: RWX 파일.
- RBD (RADOS Block Device): RWO 블록.

```bash
helm install rook-ceph rook/rook-ceph -n rook-ceph --create-namespace
# CephCluster CR 만들면 Ceph 클러스터 자동 구성
```

## 7. Volume Snapshot 깊게

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata: { name: csi-aws }
driver: ebs.csi.aws.com
deletionPolicy: Retain
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: { name: pg-snap-2026-05-18 }
spec:
  volumeSnapshotClassName: csi-aws
  source:
    persistentVolumeClaimName: pg-data
```

snapshot → 새 PVC 복원 (point-in-time recovery).

Velero가 이걸 활용해 cluster-wide backup.

## 8. Volume Cloning

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: pg-clone }
spec:
  dataSource:
    name: pg-data
    kind: PersistentVolumeClaim
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 100Gi } }
```

기존 PVC 그대로 복제. dev/staging 데이터 sync에 유용.

## 9. Generic Ephemeral Volume

Pod 수명만큼만 존재하는 임시 PVC.

```yaml
spec:
  volumes:
    - name: scratch
      ephemeral:
        volumeClaimTemplate:
          spec:
            accessModes: [ReadWriteOnce]
            resources: { requests: { storage: 10Gi } }
```

Pod 삭제 시 자동 정리. CI build runner에 유용.

## 10. 운영 함정

1. ReadWriteOnce + Deployment + replicas=3 → 2개 Pending.
2. AZ 불일치 (Immediate binding 사용 시).
3. Snapshot policy 부재 → 데이터 손실 시 회복 불가.
4. Reclaim policy Delete → PVC 삭제 시 데이터도 영구 손실. Retain 권장.
5. 큰 EBS의 attach/detach 시간 (60초+) 무시 → Pod 재배포 시 지연.

## 11. 결론

CSI 표준 덕에 어떤 스토리지든 K8s에 통합 가능. 시니어는 (1) 워크로드별 적절 storage 종류 선택, (2) snapshot/backup 운영, (3) CSI 드라이버 디버깅을 할 줄 알아야 합니다.
