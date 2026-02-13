---
layout: post
title: "OpenStack Zero-to-Hero: Week 8 - High Availability (HA) Architecture"
---

프로덕션 클라우드는 365일 24시간 죽으면 안 됩니다.
컨트롤러 노드 하나가 불타도 API는 응답해야 합니다.

오픈스택 HA의 핵심은 **"모든 것을 이중화(Redundancy)한다"**입니다.

---

## 1. VIP & HAProxy (Load Balancing)

사용자는 컨트롤러 1번의 IP(192.168.1.11)를 알 필요가 없습니다.
**VIP (Virtual IP, 192.168.1.10)**만 바라봅니다.

*   **Keepalived:** 컨트롤러 3대 중 "대장(Master)"이 VIP를 가집니다. 대장이 죽으면 부하(Backup)가 1초 만에 VIP를 가져갑니다 (VRRP).
*   **HAProxy:** VIP로 들어온 요청을 뒤에 있는 3대의 API 서버로 분산(Round Robin)합니다.

---

## 2. Database HA (Galera Cluster)

MySQL(MariaDB)을 3대에 설치하고 **Galera Cluster**로 묶습니다.
*   **Multi-Master:** 아무 데나 써도(Write) 실시간으로 다른 노드에 복제됩니다.
*   **Quorum:** 3대 중 1대가 죽어도 2대가 남아서 과반수(Quorum)를 넘으므로 정상 동작합니다. 2대가 죽으면 DB는 멈춥니다(Split-brain 방지).

---

## 3. Message Queue HA (RabbitMQ Clustering)

RabbitMQ도 3대를 클러스터링합니다.
*   **Mirrored Queues:** 큐에 들어온 메시지를 다른 노드에도 복제합니다. 노드 하나가 죽어도 메시지는 살아있습니다.

---

## 🛠️ Lab: Kill the Master

Week 7에서 구축한 멀티노드 환경(혹은 가상 환경)에서 장애 조치 테스트를 합니다.

1.  **VIP 확인:** 현재 VIP를 가진 노드를 찾습니다 (`ip a`).
2.  **Kill:** 그 노드의 전원을 끄거나 `keepalived`를 죽입니다.
3.  **Failover:** VIP가 다른 노드로 이동했는지 확인합니다.
4.  **API Test:** 그 와중에 `openstack server list` 명령어가 에러 없이 잘 되는지 봅니다.

---

## 📝 8주차 과제: Split-Brain Recovery

**목표:** Galera Cluster가 깨졌을 때(모든 노드 다운 후 재부팅 시) 복구하는 절차를 정리하세요.

1.  모든 DB 노드가 셧다운 되었습니다.
2.  다시 켰는데 DB가 안 올라옵니다. (서로 "네가 마지막 마스터야?" 하고 눈치 보는 중)
3.  `grastate.dat` 파일의 `safe_to_bootstrap: 1`을 찾아서...
4.  `galera_new_cluster` 명령어로...
5.  이 과정을 조사해서 매뉴얼로 만드세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 여러 대의 컨트롤러 노드 중 하나가 VIP(Virtual IP)를 소유하도록 관리해주는 프로토콜 및 소프트웨어는? (VRRP / Keepalived)
2.  **Q2.** MariaDB Galera Cluster에서 데이터 불일치(Split-brain)를 막기 위해 필요한 최소 노드 수는? (3개 - 홀수여야 함)
3.  **Q3.** RabbitMQ 클러스터에서 큐의 데이터를 다른 노드에도 복제하여 유실을 막는 기능은? (Mirrored Queues / HA Queues)

---

이것으로 Part 2 구축/운영 편이 끝났습니다.
이제 클라우드는 튼튼하게 돌아갑니다. 하지만 "속도"는 어떨까요?
다음 주, Part 3의 시작은 **성능 최적화(Performance Tuning)**입니다. NFV를 위한 CPU Pinning 기술을 다룹니다.

**High Availability Achieved.**
