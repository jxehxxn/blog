---
layout: post
title: "OpenStack Zero-to-Hero: Week 3 - Glance & Cinder: Image & Block Storage"
---

컴퓨터 조립할 때 윈도우 설치 USB(OS 이미지)와 하드디스크(Block Storage)가 필요하죠.
오픈스택에서는 **Glance**가 USB, **Cinder**가 하드디스크입니다.

---

## 1. Glance: Image Service

VM을 부팅할 때 쓸 OS 이미지(`ubuntu.qcow2`, `windows.iso`)를 관리합니다.

*   **Backend:** 이미지를 어디에 저장할까?
    *   **File:** 로컬 파일시스템. (`/var/lib/glance/images`) - 간단하지만 HA 구성 어려움.
    *   **Ceph (RBD):** 분산 스토리지. **(실무 표준)**. Glance가 Ceph에 저장하고, Nova도 Ceph에서 바로 읽습니다(CoW). 엄청 빠릅니다.
    *   **S3:** 오브젝트 스토리지.

---

## 2. Cinder: Block Storage Service

VM에 꽂을 **"영구적인(Persistent) 디스크"**를 제공합니다. AWS의 EBS와 같습니다.
VM이 죽어도 Cinder 볼륨은 살아있습니다.

### Architecture
*   **Cinder-API:** 요청을 받음.
*   **Cinder-Scheduler:** 어느 스토리지 노드에 볼륨을 만들지 결정.
*   **Cinder-Volume:** 실제 스토리지 장비(Dell EMC, NetApp, Ceph)와 통신해서 볼륨을 생성.

### LVM vs Ceph
*   **LVM (iSCSI):** 리눅스 서버의 로컬 디스크를 묶어서 iSCSI로 제공. 싸게 구성할 때 씀.
*   **Ceph:** 3중 복제, 자동 복구, 무제한 확장. **Cinder의 영혼의 단짝**.

---

## 3. The Boot Process (Storage View)

VM을 띄울 때 "Boot from Image"와 "Boot from Volume"의 차이를 알아야 합니다.

1.  **Boot from Image (Ephemeral):** Glance 이미지를 Compute 노드의 로컬 디스크로 복사해서 씁니다. VM 삭제하면 데이터도 날아갑니다. (웹서버 등)
2.  **Boot from Volume (Persistent):** Cinder 볼륨을 만들어서 거기에 OS를 설치하고 부팅합니다. VM 삭제해도 볼륨은 남습니다. (DB 등)

---

## 🛠️ Lab: Volume Management

1.  **이미지 등록:** `openstack image create --file cirros.img --disk-format qcow2 cirros`
2.  **볼륨 생성:** `openstack volume create --size 1 myvol`
3.  **VM 생성:** `openstack server create ...`
4.  **볼륨 연결:** `openstack server add volume myvm myvol`
5.  **확인:** VM 안에서 `fdisk -l` 해보면 새 디스크가 보입니다.

---

## 📝 3주차 과제: Cinder Backend Configuration

**목표:** Cinder가 LVM 백엔드와 Ceph 백엔드를 동시에 쓰도록 설정하는 시나리오입니다.

1.  `cinder.conf`에서 `enabled_backends = lvm, ceph` 설정.
2.  `[lvm]` 섹션과 `[ceph]` 섹션 작성.
3.  **Volume Type** 생성: `openstack volume type create lvm`, `ceph`.
4.  Type과 Backend 연결: `openstack volume type set lvm --property volume_backend_name=lvm`.
5.  사용자가 볼륨 만들 때 `--type ceph` 옵션을 주면 Ceph에 만들어지게 됩니다. 이 구조도를 그려보세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Glance 이미지를 저장하기 위한 백엔드 스토리지 중, 오픈스택과 가장 궁합이 잘 맞고 널리 쓰이는 오픈소스 분산 스토리지는? (Ceph)
2.  **Q2.** VM을 생성할 때, 'Boot from Image' 방식을 사용하면 VM이 삭제될 때 디스크 데이터는 어떻게 되나요? (삭제됨 - Ephemeral)
3.  **Q3.** Cinder에서 서로 다른 성능(SSD vs HDD)이나 백엔드 장비를 구분하여 사용자에게 선택권을 주기 위해 사용하는 논리적 개념은? (Volume Type)

---

다음 주, 오픈스택의 가장 어려운 부분, 악명 높은 **Neutron**입니다.
가상 스위치, 라우터, Floating IP의 세계로 떠납니다. 마음 단단히 먹으세요.

**Storage attached.**
