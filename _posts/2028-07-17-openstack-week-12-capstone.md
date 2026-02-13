---
layout: post
title: "OpenStack Zero-to-Hero: Week 12 - Capstone: Building a Telco-Grade Cloud"
---

축하합니다. 12주의 대장정이 끝났습니다.
이제 여러분은 오픈스택의 내부(Internal)를 꿰뚫어 보는 전문가입니다.

마지막 과제는 **"가장 까다로운 고객"**인 통신사를 위한 클라우드를 설계하는 것입니다.
통신사는 99.999%의 가용성과 마이크로초(µs) 단위의 레이턴시를 요구합니다.

---

## 🏆 Capstone Project: NFV Infrastructure Design

### 1. Requirements

가상의 통신사 "K-Telco"가 5G 코어망을 올릴 클라우드를 구축해달라고 합니다.

*   **Performance:** 패킷 처리 성능 극대화 (SR-IOV, DPDK 필수).
*   **Latency:** 지연 시간 최소화 (CPU Pinning, NUMA, Hugepages).
*   **Availability:** 컨트롤러 노드 장애 시 1초 이내 복구.
*   **Network:** Tenant 간 완벽한 격리 및 고성능 라우팅 (OVN).

### 2. Architecture Specification

**Hardware:**
*   **Controller:** 3대 (Galera, RabbitMQ, OVN Central).
*   **Compute (High Performance):** 10대 (Hugepages 1G, CPU Dedicated).
*   **Compute (General):** 5대 (Overcommit 허용).
*   **Storage:** Ceph Cluster 5대.

**Configuration (`nova.conf` & `ml2_conf.ini`):**
*   `cpu_dedicated_set` / `cpu_shared_set` 분리.
*   `mechanism_drivers = ovn`.
*   `enable_isolated_metadata = True`.

### 3. Deliverables

1.  **Architecture Diagram:** 물리/논리 네트워크 구성도.
2.  **globals.yml:** Kolla-Ansible 설정 파일 전문.
3.  **Validation Report:**
    *   `rally`를 이용한 API 성능 테스트 결과.
    *   VM 간 `iperf3` 네트워크 성능 측정 결과 (SR-IOV 적용 전후 비교).

---

## 4. Closing Remarks

오픈스택은 어렵습니다. 하지만 그만큼 **강력한 통제권**을 줍니다.
AWS에서는 할 수 없는 커널 튜닝, 네트워크 패킷 조작, 하드웨어 직접 제어가 가능합니다.

이 기술은 단순히 "오픈스택을 안다"는 것을 넘어, **"리눅스 시스템과 네트워크, 가상화의 정점"**을 이해했다는 증거입니다.

여러분이 만든 구름 위에서 세상의 데이터가 흐를 것입니다.
수고하셨습니다.

**Stack deployed successfully.**
