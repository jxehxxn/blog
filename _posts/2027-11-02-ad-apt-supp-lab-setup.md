---
layout: post
title: "Offensive AD & APT Mastery: Supplement - Building the AD Lab Environment"
---

Week 1부터 실습을 하려면 **AD Lab**이 필수입니다.
"그냥 AWS 쓰면 안 되나요?" -> 안 됩니다. AD 공격은 L2 네트워크(ARP, LLMNR)에 의존하는 경우가 많아 로컬 가상머신이 최고입니다.

이 보충 포스트에서는 **VMware** 또는 **VirtualBox**를 이용해 완벽한 실습 환경을 구축하는 법을 다룹니다.

---

## 1. Hardware Requirements

*   **CPU:** 4코어 이상 (가상화 지원 필수 - BIOS에서 VT-x 켜세요)
*   **RAM:** 16GB 권장 (최소 8GB)
*   **Disk:** SSD 100GB 이상

---

## 2. Software (ISO Images)

*   **Windows Server 2019 Evaluation:** (180일 무료)
*   **Windows 10 Enterprise Evaluation:** (90일 무료)
*   **Kali Linux:** (공격자용)

---

## 3. Network Configuration (Crucial!)

모든 VM은 인터넷과 격리된 **Host-Only Network** (또는 NAT Network)에 묶여야 합니다.

*   **Subnet:** `192.168.56.0/24` (예시)
*   **DC IP:** `192.168.56.10` (고정 IP 필수!)
*   **Client IP:** `192.168.56.20`
*   **Kali IP:** `192.168.56.100`

---

## 4. Installation Steps

### Step 1: Domain Controller (DC)
1.  Windows Server 설치.
2.  네트워크 설정에서 DNS 서버를 `127.0.0.1`로 설정.
3.  **서버 관리자 -> 역할 및 기능 추가 -> Active Directory 도메인 서비스** 설치.
4.  깃발 아이콘 클릭 -> **"이 서버를 도메인 컨트롤러로 승격"** 클릭.
5.  **새 포리스트 추가:** `corp.local` 입력.
6.  설치 후 재부팅.

### Step 2: Domain Member (Client)
1.  Windows 10 설치.
2.  네트워크 설정에서 DNS 서버를 **DC의 IP (`192.168.56.10`)**로 설정. (이거 안 하면 도메인 못 찾음)
3.  `sysdm.cpl` 실행 -> 변경 -> 소속 그룹: **도메인** -> `corp.local` 입력.
4.  Administrator ID/PW 입력하여 가입. 재부팅.

### Step 3: Populate Data (Bad Blood)
텅 빈 AD는 재미없습니다. 유저 1000명, 그룹 50개를 만들어야 합니다.
**BadBlood**라는 스크립트를 쓰면 자동으로 취약한(Bad) AD 환경을 만들어줍니다.

1.  DC에서 PowerShell 실행 (관리자 권한).
2.  `Invoke-BadBlood` 실행.
3.  커피 한 잔 마시고 오면 막장 AD가 완성되어 있습니다.

이제 여러분의 놀이터가 준비되었습니다.
