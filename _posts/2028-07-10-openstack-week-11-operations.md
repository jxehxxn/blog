---
layout: post
title: "OpenStack Zero-to-Hero: Week 11 - Upgrades & Operations (Day 2 Ops)"
---

오픈스택은 6개월마다 새 버전이 나옵니다.
업그레이드 안 하고 버티면 보안 구멍이 뚫리고, 업그레이드하다가 실패하면 서비스가 멈춥니다.
**"무중단 업그레이드(Rolling Upgrade)"**가 운영의 꽃입니다.

---

## 1. Upgrade Strategy (Kolla-Ansible)

Kolla는 업그레이드가 정말 쉽습니다.
1.  **Pull:** 새 이미지 다운로드.
2.  **Upgrade:** 컨테이너를 하나씩 내리고 새 버전으로 띄움. (Rolling)

**순서:** Keystone -> Glance -> Cinder -> Neutron -> Nova. (의존성 순서)

---

## 2. Backup & Restore

*   **Database:** `mariabackup` 도구를 사용하여 매일 밤 전체 백업(Full) + 매시간 증분 백업(Incremental).
*   **Configuration:** `/etc/kolla` 디렉토리 백업.

---

## 3. Capacity Planning

"서버를 언제 더 사야 할까요?"
*   **Overcommit Ratio 모니터링:** 물리 CPU/RAM 대비 할당된 양.
*   **Storage Usage:** Ceph 사용량이 70%를 넘으면 위험 신호. (Rebalancing 때문)

---

## 🛠️ Lab: Minor Upgrade

메이저 버전 업그레이드는 부담스러우니, 패키지 업데이트만 해봅니다.

1.  현재 `openstack-release` 확인.
2.  Kolla 이미지를 최신 태그(같은 버전 내 최신 빌드)로 바꿈.
3.  `kolla-ansible -i multinode pull`
4.  `kolla-ansible -i multinode upgrade`
5.  서비스 중단 없이 컨테이너가 교체되는지 확인.

---

## 📝 11주차 과제: Disaster Recovery Plan (DRP)

**목표:** 데이터센터 전체 정전 시 복구 계획을 수립하세요.

1.  전원 복구 후 **가장 먼저** 켜야 할 서비스는? (DB? MQ? Keystone?)
2.  Ceph 스토리지가 "Degraded" 상태일 때 VM을 켜도 될까요?
3.  Compute 노드 절반이 고장 났을 때, 중요 VM(HA)을 우선적으로 살리는 방법은? (`nova evacuate`)

---

## ✅ Self-Assessment Quiz

1.  **Q1.** OpenStack 업그레이드 시 가장 먼저 업그레이드해야 하는 핵심 서비스는? (Keystone - 다른 서비스들이 의존하므로)
2.  **Q2.** 물리 서버에 장애가 발생했을 때, 해당 서버에 있던 VM들을 다른 정상 서버로 옮겨서 재부팅하는 기능은? (Evacuate)
3.  **Q3.** 데이터베이스 백업 시 서비스 중단 없이 일관성 있는 백업을 수행하기 위해 Galera Cluster에서 사용하는 도구는? (Mariabackup 또는 XtraBackup)

---

대망의 마지막 주차입니다.
지금까지 배운 모든 것(HA, 성능 튜닝, 네트워크, 스토리지)을 집대성하여, **통신사급(Telco-Grade) NFV 클라우드**를 설계합니다.

**Day 2 Ready.**
