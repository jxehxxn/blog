---
layout: post
title: "OpenStack Zero-to-Hero: Week 7 - Deployment with Kolla-Ansible"
---

Week 1에서 DevStack을 써봤습니다. 그건 장난감입니다.
프로덕션 환경에서는 100대가 넘는 서버에 오픈스택을 깔아야 합니다. `apt-get install`로 하나씩? 불가능합니다.

**Kolla-Ansible**은 오픈스택의 모든 서비스를 **Docker 컨테이너**로 패키징하고, **Ansible**로 배포합니다.
현재 가장 널리 쓰이는 배포 방식입니다.

---

## 1. Why Containerized OpenStack?

*   **격리:** Nova 라이브러리와 Neutron 라이브러리가 충돌할 일이 없습니다. (Dependency Hell 탈출)
*   **업그레이드:** 컨테이너 이미지만 갈아끼우면 됩니다. 롤백도 쉽습니다.
*   **관리:** `docker ps`로 서비스 상태를 한눈에 봅니다.

---

## 2. Kolla-Ansible Workflow

1.  **Bootstrap:** 배포할 서버들(Inventory)에 Docker와 필요한 패키지를 깝니다.
2.  **Pre-checks:** 포트가 충돌하진 않는지, VIP가 비어있는지 검사합니다.
3.  **Pull:** Docker Hub에서 이미지를 받아옵니다.
4.  **Deploy:** 컨테이너를 띄우고 설정 파일(`nova.conf`)을 생성합니다.
5.  **Post-deploy:** `admin-openrc` 파일을 만들어줍니다.

---

## 3. Configuration (`globals.yml`)

Kolla의 모든 설정은 `/etc/kolla/globals.yml` 파일 하나에서 합니다.

```yaml
kolla_base_distro: "ubuntu"
openstack_release: "2023.2" # Bobcat
network_interface: "eth0" # 관리망
neutron_external_interface: "eth1" # 외부망
enable_cinder: "yes"
enable_heat: "no"
```

---

## 🛠️ Lab: Deploying a Multi-Node Cluster

가상머신 3대로 클러스터를 구축합니다.
*   **Node 1 (Deployer & Controller):** Ansible 실행, API 서버.
*   **Node 2 (Compute):** VM 실행.
*   **Node 3 (Storage):** Cinder/LVM.

1.  **SSH Key:** Deployer에서 모든 노드로 Passwordless SSH 설정.
2.  **Inventory:** `multinode` 파일에 IP 적기.
3.  **Deploy:**
    ```bash
    kolla-ansible -i ./multinode bootstrap-servers
    kolla-ansible -i ./multinode prechecks
    kolla-ansible -i ./multinode deploy
    ```
4.  **확인:** `docker ps`로 수십 개의 컨테이너가 뜬 걸 감상합니다.

---

## 📝 7주차 과제: Custom Config Override

**목표:** Kolla 기본 설정을 덮어쓰는 방법을 익히세요.

1.  `nova.conf`의 `cpu_allocation_ratio`를 기본 16.0에서 **4.0**으로 바꾸고 싶습니다.
2.  `/etc/kolla/config/nova/nova-compute.conf` 파일을 만들고 설정값을 적습니다.
3.  `kolla-ansible reconfigure` 명령을 실행합니다.
4.  Nova Compute 컨테이너가 재시작되고 설정이 반영되었는지 확인합니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Kolla-Ansible은 OpenStack 서비스들을 어떤 형태(Format)로 패키징하여 배포하나요? (Docker Container)
2.  **Q2.** Kolla 배포 시 전역 설정(네트워크 인터페이스, 활성화할 서비스 등)을 관리하는 메인 설정 파일의 이름은? (globals.yml)
3.  **Q3.** 배포 완료 후 설정값을 변경했을 때, 서비스를 중단하지 않고(가능한 경우) 설정을 적용하고 컨테이너를 재시작하는 명령어는? (kolla-ansible reconfigure)

---

다음 주, 서버 한 대가 죽어도 서비스는 계속되어야 합니다.
**High Availability (HA)** 구성을 파헤칩니다. VIP, HAProxy, Galera Cluster의 향연입니다.

**Container Up.**
