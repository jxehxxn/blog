---
layout: post
title: "OpenStack Zero-to-Hero: Week 10 - Troubleshooting Common Issues (RabbitMQ, DB Deadlocks)"
---

오픈스택 운영의 8할은 로그 분석입니다.
수십 개의 컴포넌트가 얽혀있어서 에러 메시지 하나 찾는 것도 일입니다.

---

## 1. The Log Chain

"VM 생성이 안 돼요."
1.  **Nova API 로그:** 요청을 받았나? (`/var/log/kolla/nova/nova-api.log`)
    *   400 Bad Request? -> 유저 잘못.
    *   500 Error? -> 서버 잘못.
2.  **Nova Scheduler 로그:** 노드를 찾았나? (`nova-scheduler.log`)
    *   "No valid host was found" -> 리소스 부족.
3.  **Nova Compute 로그:** (선택된 노드에서) 에러가 났나? (`nova-compute.log`)
    *   "LibvirtError" -> 가상화 문제.
4.  **Neutron Server 로그:** IP 할당 실패?

---

## 2. RabbitMQ Issues

가장 흔한 장애 포인트입니다.
*   **Partition:** 네트워크가 잠깐 끊겨서 노드들이 서로 "너 죽었지?" 하고 싸우는 상태.
*   **Queue Full:** 소비(Consumer)가 느려서 큐에 메시지가 수천만 개 쌓임. -> 메모리 부족 -> 전체 멈춤.

**해결:** `rabbitmqctl cluster_status` 확인 후 파티션 난 노드 재시작.

---

## 3. Database Deadlocks

Galera Cluster는 동시에 같은 행(Row)을 수정하면 **Deadlock**이 발생합니다.
*   증상: API가 느려지거나 500 에러 빈발.
*   로그: `Deadlock found when trying to get lock; try restarting transaction`.
*   해결: 보통은 재시도 로직이 있어서 괜찮지만, 너무 자주 발생하면 `haproxy` 설정을 바꿔서 Write를 한 노드로 몰아줘야 합니다.

---

## 🛠️ Lab: Breaking the Cloud

고의로 장애를 내고 고쳐봅시다.

1.  **시나리오 1:** RabbitMQ 컨테이너 1개 강제 종료. -> 클러스터 상태 확인 및 서비스 영향도 체크.
2.  **시나리오 2:** Compute 노드의 디스크 용량을 꽉 채움 (`dd if=/dev/zero of=bigfile`). -> VM 생성 시도 -> `No valid host` 에러 확인.
3.  **시나리오 3:** `nova.conf`에 오타 내고 컨테이너 재시작. -> 로그(`docker logs nova_compute`) 보고 오타 찾기.

---

## 📝 10주차 과제: Root Cause Analysis (RCA)

**목표:** 과거의 장애 사례를 분석하여 RCA 보고서를 쓰세요. (가상)

*   **증상:** 월요일 아침 9시에 전사 직원이 로그인하자 Keystone 응답이 30초 지연됨.
*   **원인:** (가설) Token 테이블에 만료된 토큰이 1억 개 쌓여서 DB 조회가 느려짐.
*   **해결:** `keystone-manage token_flush` 크론잡이 실패하고 있었음. 재실행 및 인덱스 튜닝.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Nova Scheduler가 "No valid host was found" 에러를 뱉는 가장 흔한 원인 두 가지는? (리소스 부족, Flavor 조건 불일치)
2.  **Q2.** RabbitMQ 클러스터가 네트워크 단절로 인해 쪼개지는 현상을 무엇이라 하나요? (Network Partition)
3.  **Q3.** MySQL Galera Cluster에서 동시에 같은 데이터에 쓰기를 시도할 때 발생하는 에러는? (Deadlock)

---

다음 주, "새 버전 나왔는데 어떻게 올려요?"
중단 없는 **업그레이드**와 일상적인 운영 업무를 배웁니다.

**Problem Solved.**
