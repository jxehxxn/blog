---
layout: post
title: "OpenStack Zero-to-Hero: Week 9 - Performance Tuning (CPU Pinning, Hugepages)"
---

"VM은 물리 서버보다 느리다."
옛날 말입니다. **CPU Pinning**과 **Hugepages**를 쓰면, VM도 물리 서버 성능의 98%까지 낼 수 있습니다. (Telco/NFV의 필수 요건)

---

## 1. NUMA (Non-Uniform Memory Access)

요즘 서버는 CPU 소켓이 2개입니다.
CPU 0번은 RAM 0번 슬롯에 가깝고, CPU 1번은 RAM 1번에 가깝습니다.
*   **Local Access:** CPU 0 -> RAM 0 (빠름)
*   **Remote Access:** CPU 0 -> CPU 1 -> RAM 1 (느림, 병목)

오픈스택 스케줄러는 이걸 고려해서 **"VM이 CPU 0을 쓰면 메모리도 RAM 0에서 할당"**해줘야 합니다. (`NUMATopologyFilter`)

---

## 2. CPU Pinning (vCPU to pCPU Mapping)

보통은 vCPU가 물리 코어 사이를 왔다 갔다 합니다(Context Switch).
**Pinning**은 "이 VM의 vCPU 0번은 무조건 물리 코어 4번만 써!"라고 못 박는 겁니다. 캐시 적중률이 올라가고 지연 시간(Latency)이 사라집니다.

```bash
# nova.conf
[compute]
cpu_dedicated_set = 2-15,18-31
cpu_shared_set = 0-1,16-17
```

---

## 3. Hugepages

리눅스 메모리는 4KB 단위로 관리됩니다. 100GB 메모리를 관리하려면 페이지 테이블이 너무 커집니다.
**Hugepages** (2MB 또는 1GB)를 쓰면 페이지 테이블 크기가 줄어들어 메모리 접근 속도가 빨라집니다.

*   Flavor 설정: `hw:mem_page_size=1GB`

---

## 4. SR-IOV & OVS-DPDK

네트워크 패킷 처리를 커널이 아닌 **유저 공간(DPDK)**에서 하거나, 아예 **NIC 하드웨어(SR-IOV)**가 직접 VM에 꽂히게 합니다.
패킷 처리량이 10배 이상 뜁니다.

---

## 🛠️ Lab: Creating a High-Performance VM

1.  **Host 설정:** 커널 파라미터에 `default_hugepagesz=1G hugepagesz=1G hugepages=4` 등을 추가하고 재부팅.
2.  **Flavor 생성:**
    ```bash
    openstack flavor create --ram 4096 --disk 20 --vcpus 2 nfv.large
    openstack flavor set nfv.large --property hw:cpu_policy=dedicated
    openstack flavor set nfv.large --property hw:mem_page_size=large
    ```
3.  **VM 생성:** 위 Flavor로 VM 생성.
4.  **확인:** Compute 노드에서 `virsh dumpxml instance-000xxx`를 보면 `<vcpupin>` 태그가 보입니다.

---

## 📝 9주차 과제: NUMA Awareness

**목표:** 듀얼 소켓 서버에서 "NUMA 균형"을 맞추는 전략을 세우세요.

1.  Socket 0에만 VM이 몰리면 Socket 1의 자원은 놀게 됩니다.
2.  Flavor A는 "Socket 0 선호", Flavor B는 "Socket 1 선호"하도록 설정하려면 `hw:numa_nodes`와 같은 속성을 어떻게 써야 할까요?
3.  PCI Passthrough(GPU 등) 장치를 쓸 때 NUMA가 중요한 이유를 서술하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** VM의 vCPU를 특정 물리 코어에 고정하여 컨텍스트 스위칭 비용을 줄이는 기술은? (CPU Pinning)
2.  **Q2.** 대용량 메모리를 사용하는 VM의 페이지 테이블 조회 성능을 높이기 위해 사용하는 리눅스 커널 기능은? (Hugepages)
3.  **Q3.** 네트워크 패킷 처리 시 커널의 오버헤드를 우회하기 위해 사용자 공간(User Space)에서 직접 패킷을 폴링(Polling)하는 기술은? (DPDK - Data Plane Development Kit)

---

다음 주, "DB가 멈췄어요", "RabbitMQ가 터졌어요."
운영자의 악몽, **장애 처리(Troubleshooting)** 노하우를 전수합니다.

**Turbo Boosted.**
