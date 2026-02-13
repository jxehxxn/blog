---
layout: post
title: "OpenStack Zero-to-Hero: Supplement - Linux Networking for OpenStack (Namespaces, Bridges)"
---

Neutron이 마법을 부리는 것 같지만, 사실은 리눅스 커널 기능을 교묘하게 엮어놓은 것입니다.
이 기본기를 모르면 `ip a` 쳤을 때 나오는 수많은 인터페이스들을 보고 현기증을 느끼게 됩니다.

---

## 1. Network Namespaces (netns)

리눅스 안에 또 다른 리눅스를 만드는 기술입니다. (Docker의 원리이기도 하죠.)
Neutron은 **가상 라우터 하나마다 네임스페이스 하나**를 만듭니다.

*   `ip netns list`: 네임스페이스 목록 보기 (`qrouter-xxx`, `qdhcp-xxx`)
*   `ip netns exec qrouter-xxx ip a`: 그 라우터 안의 인터페이스 보기.
*   **핵심:** 라우터 네임스페이스 안에는 자신만의 라우팅 테이블, Iptables 규칙이 따로 있습니다. 그래서 테넌트 A의 10.0.0.1과 테넌트 B의 10.0.0.1이 겹쳐도 문제없는 것입니다(**IP Overlap**).

---

## 2. Linux Bridge vs OVS Bridge

*   **Linux Bridge:** 단순한 L2 스위치. Security Group(방화벽)을 구현하기 위해 Iptables와 연동하기 쉬워서 예전에 많이 썼습니다.
*   **OVS Bridge:** 프로그래머블 스위치. OpenFlow 규칙을 통해 패킷을 아주 세밀하게 제어(VLAN 변경, 터널링, 미러링)할 수 있습니다. Neutron의 주력입니다.

---

## 3. veth pairs (Virtual Ethernet Cable)

네임스페이스와 네임스페이스, 혹은 브리지와 네임스페이스를 연결하는 **가상 랜선**입니다.
항상 쌍(Pair)으로 생성됩니다. A 끝단에 패킷을 넣으면 B 끝단으로 나옵니다.

Neutron에서 `ip a`를 보면 `tap`, `qvb`, `qvo` 같은 이름이 보입니다. 이게 다 veth pair의 한쪽 끝입니다.

---

## 4. The Packet Flow (Visualized)

VM -> `tap` (Firewall) -> Linux Bridge -> `qvb` --(veth)--> `qvo` -> OVS Bridge (`br-int`) -> ...

이 복잡한 연결 고리를 이해해야 "VM이 통신이 안 돼요" 할 때 어디가 끊어졌는지 찍어볼 수 있습니다.
