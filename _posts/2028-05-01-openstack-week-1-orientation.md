---
layout: post
title: "OpenStack Zero-to-Hero: Week 1 - Orientation & Architecture"
---

안녕하세요, 클라우드 엔지니어 여러분.

AWS, Azure 같은 퍼블릭 클라우드가 대세인 세상이지만, 여전히 통신사, 금융권, 대기업, 연구소는 **프라이빗 클라우드(Private Cloud)**를 필요로 합니다. 그리고 그 프라이빗 클라우드의 사실상 표준(De facto standard)이 바로 **OpenStack**입니다.

하지만 OpenStack은 악명 높습니다. "어렵다", "복잡하다", "툭하면 깨진다".
맞습니다. OpenStack은 단순한 소프트웨어가 아니라, 수십 개의 오픈소스 프로젝트가 맞물려 돌아가는 **거대한 OS**와 같기 때문입니다.

이 12주 과정은 단순한 설치 가이드가 아닙니다. **"왜 이렇게 설계되었는가?"**를 파고들어, 문제가 생겼을 때 로그 한 줄만 보고도 원인을 파악할 수 있는 **진짜 전문가(Hero)**를 양성하는 것이 목표입니다.

---

## 📅 12주 완성 커리큘럼 (Syllabus)

우리는 밑바닥부터 시작해서 성능 최적화(Performance Tuning)까지 올라갈 것입니다.

### Part 1: Core Components (기초 다지기)
*   **W1:** 오리엔테이션 & 아키텍처 개요 (Big Picture)
*   **W2:** Keystone (인증의 관문)
*   **W3:** Glance & Cinder (스토리지의 모든 것)
*   **W4:** Neutron (네트워크 가상화의 심장 - OVS/OVN)
*   **W5:** Nova (컴퓨팅 가상화의 핵심)

### Part 2: Deployment & Operations (구축과 운영)
*   **W6:** Management Interfaces (Horizon, CLI, SDK)
*   **W7:** Kolla-Ansible (컨테이너 기반 배포의 정석)
*   **W8:** High Availability (HA) Architecture (RabbitMQ, Galera Cluster)

### Part 3: Expert Tuning & Troubleshooting (심화)
*   **W9:** Performance Tuning (CPU Pinning, NUMA, Hugepages) - *가장 중요!*
*   **W10:** Troubleshooting (로그 분석, RPC 타임아웃, DB 데드락)
*   **W11:** Day 2 Operations (업그레이드, 백업, 확장)
*   **W12:** Capstone - Telco-Grade (NFV) 클라우드 구축

---

## 🎓 1주차 강의: The Big Picture

### 1. What is OpenStack?
오픈스택은 **"데이터센터의 자원(CPU, RAM, Disk, Network)을 API로 제어하게 해주는 운영체제"**입니다.
서버실에 가서 랜선을 꽂는 대신, `openstack server create` 명령어를 치면 VM이 뚝딱 만들어지는 마법을 부립니다.

### 2. Logical Architecture
오픈스택은 수많은 **서비스(Service)**들의 집합입니다. 각 서비스는 서로 **REST API**와 **Message Queue(RabbitMQ)**로 대화합니다.

*   **Keystone:** "나 누구야?" (Identity)
*   **Nova:** "VM 만들어줘." (Compute)
*   **Neutron:** "IP 주고 네트워크 연결해줘." (Networking)
*   **Cinder:** "하드디스크 붙여줘." (Block Storage)
*   **Glance:** "OS 이미지 줘." (Image Service)
*   **Swift:** "파일 저장해줘." (Object Storage)

이 서비스들이 유기적으로 연결되어 하나의 거대한 시스템을 이룹니다. 하나라도 아프면 전체가 비틀거립니다.

### 3. Physical Architecture
실제 하드웨어는 어떻게 구성될까요?
*   **Controller Node:** 두뇌. API 서버, DB, MQ가 돌아갑니다. (보통 3대 HA 구성)
*   **Compute Node:** 몸체. 실제 VM이 뜨는 곳입니다. (수십~수백 대)
*   **Storage Node:** 창고. Cinder/Swift가 스토리지를 제공합니다. (Ceph 등 사용)

---

## 🛠️ Lab: DevStack Setup

첫 주니까 가볍게 **DevStack**으로 맛보기만 하겠습니다. (프로덕션용 아님!)

### 1. VM 준비 (Ubuntu 22.04)
*   CPU: 4 Core+
*   RAM: 8GB+ (16GB 권장)
*   Disk: 50GB+

### 2. DevStack 설치
```bash
git clone https://opendev.org/openstack/devstack
cd devstack
# local.conf 작성 (비밀번호 설정)
./stack.sh
```
설치가 끝나면 대시보드 IP가 나옵니다.

### 3. 맛보기
*   `source openrc`
*   `openstack service list`: 서비스들이 살아있는지 확인.
*   `openstack server create`: VM 하나 띄워보기.

---

## 📝 1주차 과제: Architecture Diagram

**목표:** 오픈스택 컴포넌트 간의 통신 흐름을 이해하세요.

1.  사용자가 `openstack server create` 명령을 내렸을 때,
2.  **Keystone -> Nova API -> Nova Scheduler -> Nova Compute -> Glance -> Neutron -> Cinder** 로 이어지는 호출 흐름을 다이어그램으로 그리세요.
3.  각 단계에서 **REST API**를 쓰는지, **RPC(Message Queue)**를 쓰는지 표시하세요. (이게 트러블슈팅의 핵심입니다!)

---

## ✅ Self-Assessment Quiz

1.  **Q1.** OpenStack 서비스들 간의 인증 토큰을 발급하고 검증하는 서비스의 이름은? (Keystone)
2.  **Q2.** OpenStack 컴포넌트들이 비동기 통신을 위해 사용하는 기본 메시지 큐 시스템은? (RabbitMQ)
3.  **Q3.** 실제 가상머신(VM) 인스턴스가 생성되고 실행되는 물리적 노드의 역할(Role) 이름은? (Compute Node)

---

다음 주, 오픈스택의 모든 요청이 거쳐가는 첫 번째 관문, **Keystone**을 뜯어봅니다.
토큰이 만료되면 아무것도 할 수 없습니다.

**Stack on.**
