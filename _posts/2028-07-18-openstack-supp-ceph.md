---
layout: post
title: "OpenStack Zero-to-Hero: Supplement - Ceph Storage Operations & Tuning"
---

"OpenStack이 멈췄어요."
알고 보면 십중팔구 **Ceph** 문제입니다. 디스크 하나가 죽어서 복구(Recovery)하느라 전체 I/O가 느려졌거나, PG(Placement Group) 설정이 꼬인 경우죠.

Ceph를 모르면 OpenStack을 반쪽밖에 모르는 것입니다. 이 보충 강의에서는 **Ceph를 OpenStack의 관점**에서 운영하고 튜닝하는 실전 기법을 다룹니다.

---

## 1. Ceph Architecture Recap

*   **OSD (Object Storage Daemon):** 디스크 하나당 OSD 프로세스 하나. 데이터 저장꾼.
*   **MON (Monitor):** 클러스터 지도(Cluster Map)를 관리하는 두뇌. (3대 필수)
*   **MGR (Manager):** 대시보드 및 통계 수집.
*   **Pool:** 데이터를 담는 논리적 공간. (OpenStack은 `vms`, `volumes`, `images` 풀을 사용)

---

## 2. The CRUSH Map

Ceph의 마법입니다. 중앙 DB 없이 계산만으로 "이 데이터가 어느 디스크에 있는지" 압니다.
하지만 기본 설정은 위험합니다.

*   **Failure Domain:** 기본은 `host`입니다. 서버 한 대가 죽어도 데이터는 안전합니다.
    *   **Tuning:** 랙(Rack) 단위로 설정하면, 랙 전체 전원이 나가도 안전하게 만들 수 있습니다.
*   **Device Class:** SSD와 HDD가 섞여 있다면?
    *   `vms` 풀(고성능)은 SSD에만, `images` 풀(저장용)은 HDD에만 저장되도록 CRUSH Rule을 만들어야 합니다.

```bash
ceph osd crush rule create-replicated rule-ssd default host ssd
ceph osd pool set vms crush_rule rule-ssd
```

---

## 3. PG Calculation & Tuning

**Placement Group (PG)** 개수는 성능의 핵심입니다.
*   너무 적으면: 데이터 분포가 불균형해짐.
*   너무 많으면: OSD 메모리 폭발 및 피어링 속도 저하.

**Golden Rule:**
`Total PGs = (OSD 수 * 100) / Replica 수`
그리고 `pg_num`은 2의 제곱수여야 합니다. (예: 128, 256, 512...)

---

## 4. Recovery Operations

디스크가 고장 나서 새 디스크로 교체했습니다. Ceph는 데이터를 복구(Rebalance)하기 시작합니다.
이때 **서비스 성능이 급격히 떨어집니다.** 복구 트래픽이 서비스 트래픽을 잡아먹기 때문입니다.

**운영 팁:** 업무 시간에는 복구 속도를 늦추고, 밤에 올리세요.

```bash
# 낮 (서비스 우선)
ceph tell 'osd.*' injectargs --osd_max_backfills=1
ceph tell 'osd.*' injectargs --osd_recovery_max_active=1

# 밤 (복구 우선)
ceph tell 'osd.*' injectargs --osd_max_backfills=16
ceph tell 'osd.*' injectargs --osd_recovery_max_active=4
```

---

## 5. Client Integration (Libvirt Secret)

Nova가 Ceph를 쓰려면 **Libvirt Secret** (인증 키)가 필요합니다.
`nova.conf`와 `cinder.conf`에 있는 `rbd_secret_uuid`가 실제 Libvirt에 등록된 Secret UUID와 일치해야 합니다.

```bash
virsh secret-list
virsh secret-get-value <UUID>
```
이게 틀리면 "VM 생성 중 Block Device Mapping 실패" 에러가 뜹니다.

---

## Summary

Ceph는 "설치하면 끝"이 아니라 **"살아있는 생물"**입니다.
상태(`ceph -s`)가 `HEALTH_OK`가 아니면 퇴근하지 마세요.
`pg_num`을 계산하고, 복구 속도를 조절하는 엔지니어가 진짜 전문가입니다.
