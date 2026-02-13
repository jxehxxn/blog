---
layout: post
title: "OpenStack Zero-to-Hero: Supplement - Network Packet Debugging with tcpdump & Wireshark"
---

"Ping이 안 나가요."
초보자는 Security Group을 봅니다.
중수는 라우팅 테이블을 봅니다.
**고수는 패킷을 깝니다.**

오픈스택 네트워크는 복잡한 터널(VXLAN/Geneve) 속에 숨어있습니다.
이걸 뜯어보고(Decapsulation) 범인을 잡는 법을 배웁니다.

---

## 1. The Packet Journey (Trace Point)

VM A (10.0.0.5)에서 VM B (10.0.0.6)로 핑을 쏠 때, 패킷은 어디를 지날까요?

1.  **VM A Interface (`eth0`):** 출발.
2.  **Compute Node A (`tapXXXX`):** 하이퍼바이저로 진입.
3.  **OVS Bridge (`br-int`):** VLAN 태깅.
4.  **Physical Interface (`eth1`):** **캡슐화(Encapsulation)**되어 나감. (여기선 10.0.0.x IP가 안 보임!)
5.  **Compute Node B (`eth1`):** 패킷 도착.
6.  **...**

---

## 2. Decapsulation with Wireshark

물리 인터페이스(`eth1`)에서 덤프를 뜨면 외계어가 보입니다.
`Outer IP: 192.168.1.10 -> 192.168.1.20, UDP 6081 (Geneve)`

내부 패킷(Inner Packet)을 보려면 Wireshark 설정이 필요합니다.
*   **Edit -> Preferences -> Protocols -> Geneve (또는 VXLAN)**
*   UDP Port를 `6081` (Geneve) 또는 `4789` (VXLAN)로 설정.

이제 내부의 `10.0.0.5 -> 10.0.0.6 ICMP` 패킷이 보입니다!

---

## 3. Tcpdump on the Command Line

GUI 없이 서버에서 바로 확인하려면?

```bash
# VXLAN 내부 헤더까지 보기 (vni 필터링)
tcpdump -i eth1 -nne udp port 4789

# 특정 VM의 tap 인터페이스만 보기 (가장 확실함)
tcpdump -i tap123456-78 -nne
```

**꿀팁:** `-e` 옵션을 줘서 **MAC Address**를 꼭 확인하세요.
"IP는 맞는데 MAC이 틀리네?" -> **ARP Spoofing**이나 **IP 중복** 이슈입니다.

---

## 4. OVN Trace (`ovn-trace`)

OVN을 쓴다면 `tcpdump`보다 더 강력한 도구가 있습니다.
"패킷이 논리적으로 어떻게 흘러갈지"를 시뮬레이션해줍니다.

```bash
# VM A에서 VM B로 가는 패킷 시뮬레이션
ovn-trace --ct next --vm <VM_A_UUID> <VM_B_IP>
```
결과: `ingest -> acl (allow) -> routing -> output to <VM_B_UUID>`
이 로그를 보면 **어느 ACL(방화벽)에서 드랍되었는지** 정확히 알 수 있습니다.

---

## 5. Summary

*   **Security Group 문제:** `tap` 인터페이스에서 패킷이 나가는데 응답이 안 오면 SG 문제.
*   **MTU 문제:** 핑은 가는데 SSH가 안 된다? `tcpdump`로 패킷 사이즈 확인. (터널 헤더 때문에 MTU 1450으로 낮춰야 함)
*   **라우팅 문제:** `ovn-trace`로 논리적 흐름 확인.

네트워크는 거짓말을 하지 않습니다. 패킷이 진실입니다.
