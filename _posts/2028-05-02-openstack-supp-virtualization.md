---
layout: post
title: "OpenStack Zero-to-Hero: Supplement - Virtualization Fundamentals (KVM/QEMU)"
---

OpenStack Nova를 배우기 전에, 그 아래에 있는 **가상화(Virtualization)** 기술을 모르면 껍데기만 배우는 것입니다.
"Nova가 VM을 만든다"고 하지만, 실제로 만드는 건 **KVM**과 **QEMU**입니다. Nova는 단지 지시만 할 뿐이죠.

이 보충 강의에서는 리눅스 가상화의 핵심을 찌릅니다.

---

## 1. Hypervisors: Type 1 vs Type 2

*   **Type 1 (Bare-metal):** 하드웨어 위에 바로 하이퍼바이저가 올라갑니다. (ESXi, Xen)
*   **Type 2 (Hosted):** OS 위에 프로그램처럼 돌아갑니다. (VirtualBox, VMware Workstation)

**KVM (Kernel-based Virtual Machine)**은 독특합니다. 리눅스 커널 자체를 하이퍼바이저로 만들어버립니다. 그래서 성능이 끝내줍니다. (Type 1에 가깝게 동작)

---

## 2. KVM vs QEMU vs Libvirt

이 셋의 관계를 헷갈리는 사람이 많습니다.

*   **KVM:** CPU와 메모리를 가상화하는 **커널 모듈**. "CPU야, 이 프로세스는 VM이니까 간섭하지 마."
*   **QEMU:** 디스크, 네트워크 카드, 키보드 같은 **I/O 장치**를 에뮬레이션하는 **유저 공간 프로세스**. KVM의 도움을 받아 CPU 속도를 높임.
*   **Libvirt:** KVM/QEMU를 쉽게 다루게 해주는 **API 및 도구 (`virsh`)**. OpenStack Nova는 QEMU 명령어를 직접 치지 않고 Libvirt API를 호출합니다.

**결론:** Nova -> Libvirt -> QEMU -> KVM -> Hardware

---

## 3. Storage Virtualization

VM은 디스크를 어떻게 볼까요? 파일(`disk.qcow2`)로 봅니다.
*   **QCOW2 (QEMU Copy On Write):** 얇은 프로비저닝(Thin Provisioning). 처음엔 작다가 데이터 쓸 때만 커짐. 스냅샷 가능.
*   **RAW:** 그냥 쌩 파일. 빠르지만 공간 낭비.

OpenStack에서는 보통 Glance에서 이미지를 받아와서 Compute 노드의 `/var/lib/nova/instances/` 아래에 QCOW2 파일로 저장하고 VM을 띄웁니다.

---

## 4. Troubleshooting Tip

Nova가 "VM 생성 실패"라고 에러를 뱉으면, Nova 로그만 보지 말고 **Libvirt 로그**를 봐야 합니다.
`/var/log/libvirt/qemu/instance-0000001.log`

여기엔 QEMU 프로세스가 뱉은 찐 에러("No bootable device", "Permission denied")가 들어있습니다.

---

## 5. Summary

*   OpenStack은 마법사가 아닙니다. Libvirt와 QEMU를 부리는 **지휘자**일 뿐입니다.
*   성능 이슈(CPU Steal 등)가 생기면 KVM 레벨을 봐야 하고, I/O가 느리면 QEMU 설정을 봐야 합니다.

이 기본기가 있어야 Week 5(Nova)와 Week 9(Performance Tuning)를 소화할 수 있습니다.
