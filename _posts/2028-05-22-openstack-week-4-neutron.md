---
layout: post
title: "OpenStack Zero-to-Hero: Week 4 - Neutron: Networking Architecture (OVS/OVN)"
---

오픈스택이 어렵다면 그건 90% 확률로 **Neutron** 때문입니다.
물리 네트워크 위에 가상 네트워크(Overlay)를 깔고, 라우팅, 방화벽, 로드밸런싱까지 다 소프트웨어로 처리합니다.

이전에는 **OVS (Open vSwitch)** + Linux Bridge 조합을 썼지만, 최근 대세는 **OVN (Open Virtual Network)**입니다. 우리는 이 두 가지를 모두 봅니다.

---

## 1. Network Types: Provider vs Self-Service

*   **Provider Network:** 데이터센터의 물리 네트워크(VLAN)를 그대로 VM에 연결합니다. 관리자가 만듭니다. (공인 IP 바로 할당)
*   **Self-Service Network (Tenant Network):** 사용자가 맘대로 만드는 가상 네트워크(VXLAN/Geneve). 물리 스위치 설정 필요 없음. (사설 IP)

---

## 2. OVS Architecture (Legacy but Common)

패킷 하나가 VM에서 밖으로 나갈 때 거치는 과정이 험난합니다.

1.  **VM** -> Tap Device
2.  **Linux Bridge** (Security Group 처리)
3.  **OVS Integration Bridge (`br-int`)**
4.  **OVS Tunnel Bridge (`br-tun`)** -> VXLAN 캡슐화
5.  **Physical Interface (`eth0`)** -> 밖으로

너무 복잡해서 트러블슈팅이 힘들고 성능도 떨어집니다.

---

## 3. OVN Architecture (The Future)

OVN은 OVS를 더 똑똑하게 제어하는 시스템입니다.
*   **Distributed Router:** 모든 Compute 노드에 라우터가 분산되어 있습니다. (East-West 트래픽 최적화)
*   **No Agents:** Python 에이전트 대신 OVN Controller가 직접 OVS를 제어합니다. 빠르고 가볍습니다.
*   **Database Driven:** `Northbound DB`(논리적 설정)와 `Southbound DB`(물리적 상태)를 사용하여 상태를 동기화합니다. RabbitMQ 의존성을 없앴습니다!

---

## 4. Floating IP & SNAT

사설 IP를 가진 VM이 인터넷에 나가려면?
*   **SNAT:** 나갈 때 라우터(Network Node)의 공인 IP로 바꿔치기.
*   **Floating IP (DNAT):** 밖에서 들어올 때 공인 IP를 사설 IP로 바꿔치기.

---

## 🛠️ Lab: Packet Walk

VM A에서 VM B로 핑을 쏠 때, 패킷이 어디를 거쳐가는지 `tcpdump`로 찍어봅니다.

1.  **Compute Node:** `tcpdump -i tapXXXX` (VM 인터페이스)
2.  **Network Node:** `ip netns exec qrouter-XXXX tcpdump` (라우터 네임스페이스)
3.  **Encapsulation:** `tcpdump -i eth0 udp port 4789` (VXLAN 패킷 확인)

---

## 📝 4주차 과제: OVS Flow Analysis

**목표:** `ovs-ofctl` 명령어로 OpenFlow 규칙을 해석하세요.

1.  `ovs-ofctl dump-flows br-int` 명령을 실행합니다.
2.  **Table 0 (Classifier):** 패킷을 분류하는 규칙을 찾으세요.
3.  **VLAN Tagging:** 로컬 VLAN ID를 붙이는 규칙을 찾으세요.
4.  이 규칙들이 어떻게 VM간 통신을 격리(Isolation)하는지 설명하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** 물리 네트워크 장비 설정 없이 가상 네트워크(Overlay)를 만들기 위해 사용하는 캡슐화 프로토콜 두 가지는? (VXLAN, Geneve)
2.  **Q2.** OVN 아키텍처에서 논리적 네트워크 설정과 물리적 네트워크 상태 정보를 저장하는 두 가지 데이터베이스는? (Northbound DB, Southbound DB)
3.  **Q3.** 리눅스 커널의 네트워크 스택(IP, Route, Iptables 등)을 논리적으로 완전히 격리하여 가상 라우터 등을 구현하는 기술은? (Network Namespace - netns)

---

다음 주, 이 모든 인프라 위에서 실제로 CPU를 쪼개고 VM을 실행하는 **Nova**를 배웁니다.
가상화의 끝판왕입니다.

**Network connected.**
