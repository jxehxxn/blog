---
layout: post
title: "OpenStack Zero-to-Hero: Week 5 - Nova: Compute Service Internals"
---

오픈스택의 존재 이유, **Nova**입니다.
사용자 요청을 받아서, 적절한 Compute 노드를 고르고, 네트워크와 스토리지를 붙여서 VM을 탄생시키는 총괄 지휘자입니다.

---

## 1. Nova Components

*   **Nova-API:** 유저 요청을 받음.
*   **Nova-Scheduler:** "이 VM을 어디에 띄울까?" (빈 공간 찾기 - Filtering & Weighing).
*   **Nova-Conductor:** DB 접근을 대행해주는 중재자. (보안상의 이유로 Compute 노드는 DB에 직접 접근 못 함).
*   **Nova-Compute:** (각 노드에 존재) 실제 하이퍼바이저(Libvirt)를 제어해서 VM 생성.

---

## 2. The Scheduling Process (Filter & Weight)

스케줄러가 노드를 고르는 과정은 두 단계입니다.

1.  **Filtering:** 조건에 안 맞는 애들 탈락.
    *   `RamFilter`: 램 부족한 노드 탈락.
    *   `ComputeFilter`: 죽은 노드 탈락.
    *   `AvailabilityZoneFilter`: 다른 AZ 노드 탈락.
2.  **Weighing:** 남은 애들 중에 점수 매기기.
    *   `RamWeigher`: 램이 가장 많이 남은 놈이 1등. (Spread 전략)

---

## 3. Instance Lifecycle

VM의 상태 변화를 알아야 합니다.
*   **Build:** 생성 중 (이미지 다운로드, 네트워킹 설정).
*   **Active:** 정상 실행 중.
*   **Shutoff:** 전원 꺼짐.
*   **Shelved:** 장기 휴면. (하이퍼바이저 리소스 해제, 이미지는 스토리지로 대피).
*   **Error:** 망함. (로그 확인 필수)

---

## 4. Resource Overcommit

가상화의 꽃입니다. 물리 자원보다 더 많은 자원을 할당하는 것입니다.
*   **CPU Allocation Ratio:** 16.0 (물리 코어 1개당 vCPU 16개 할당 가능). 대부분의 VM은 CPU를 100% 안 쓰니까요.
*   **RAM Allocation Ratio:** 1.5 (물리 램보다 1.5배 할당).

하지만 **Telco/NFV** 환경에서는 오버커밋을 끕니다(Ratio 1.0). 성능 보장을 위해서죠.

---

## 🛠️ Lab: Manual Scheduling

스케줄러를 무시하고 특정 호스트에 VM을 강제로 띄워봅니다.

1.  **Availability Zone 생성:** `openstack aggregate create --zone myzone my-aggregate`
2.  **호스트 추가:** `openstack aggregate add host my-aggregate compute-01`
3.  **강제 배포:** `openstack server create --availability-zone myzone:compute-01 ...`
4.  `openstack server show`로 `OS-EXT-SRV-ATTR:host`가 맞는지 확인.

---

## 📝 5주차 과제: Scheduler Tuning

**목표:** 고성능 VM을 위한 스케줄러 설정 (`nova.conf`)을 설계하세요.

1.  **NUMATopologyFilter**를 켜서 NUMA 아키텍처를 고려하게 하세요.
2.  **RamWeigher** 대신, CPU 사용량이 적은 노드를 선호하도록 가중치 설정을 바꾸세요.
3.  **ServerGroupAntiAffinityFilter**를 사용하여, 웹 서버 2대가 절대 같은 물리 노드에 뜨지 않게(HA) 하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Nova 컴포넌트 중, Compute 노드들이 데이터베이스에 직접 접근하는 것을 막고 DB 작업을 대행해주는 역할을 하는 것은? (Nova-Conductor)
2.  **Q2.** Nova Scheduler가 적절한 호스트를 선정할 때 수행하는 두 가지 단계는? (Filtering -> Weighing)
3.  **Q3.** 가상화 환경에서 물리 자원보다 논리적으로 더 많은 자원을 할당하여 효율성을 높이는 기술을 무엇이라 하나요? (Overcommit)

---

이것으로 Part 1: 핵심 컴포넌트가 끝났습니다.
이제 오픈스택의 뼈대는 알았습니다. 다음 주부터는 이걸 **어떻게 설치하고 운영할지(Part 2)**를 배웁니다. 손쉬운 배포의 구세주, **Kolla-Ansible**이 기다립니다.

**VM Active.**
