---
layout: post
title: "OpenStack Zero-to-Hero: Week 2 - Keystone: Identity Service Deep Dive"
---

오픈스택의 모든 API 요청은 **Keystone**으로 시작해서 Keystone으로 끝납니다.
"VM 만들어줘"라고 Nova에게 말하면, Nova는 제일 먼저 "너 누구야? 권한 있어?"라고 Keystone에게 물어봅니다.

Keystone이 죽으면 오픈스택 전체가 마비됩니다. 가장 중요합니다.

---

## 1. Key Concepts: User, Project, Role

*   **User:** 사람 또는 서비스 계정(Nova, Neutron 등).
*   **Project (Tenant):** 자원 격리의 단위. (AWS의 VPC나 Account 개념). "HR팀 프로젝트", "Dev팀 프로젝트".
*   **Role:** 권한. "admin", "member", "reader".
*   **Assignment:** User + Project + Role. "철수는 HR팀 프로젝트의 admin이다."

---

## 2. Authentication & Token

로그인을 하면 Keystone은 **Token**을 줍니다. (Fernet Token).
이 토큰은 "신분증"입니다. 다른 서비스에 API를 호출할 때 헤더(`X-Auth-Token`)에 넣어서 보냅니다.

*   **Fernet Token:** DB에 저장되지 않는 암호화된 토큰. (예전엔 UUID 토큰을 DB에 저장해서 성능 병목이 심했음)
*   **Token Validation:** Nova가 요청을 받으면, 이 토큰이 유효한지 Keystone에게 물어봅니다(Validate). 매 요청마다 물어보면 느리니까 **Memcached**에 캐싱합니다.

---

## 3. Catalog Service

Keystone은 **전화번호부** 역할도 합니다.
"Nova API 주소가 뭐지?", "Glance는 어디 있지?"
클라이언트는 Keystone에게서 **Service Catalog**를 받아와서 각 서비스의 엔드포인트 URL을 찾습니다.

*   **Endpoints:** Public (인터넷용), Internal (내부 서비스용), Admin (관리자용).

---

## 🛠️ Lab: CLI Practice

OpenStack CLI로 인증 구조를 파헤쳐 봅니다.

1.  **Token 발급:** `openstack token issue`. 내 토큰 값을 확인합니다.
2.  **Catalog 확인:** `openstack catalog list`. 서비스들의 URL을 확인합니다.
3.  **User 생성:** `openstack user create --password secret myuser`.
4.  **Project 생성:** `openstack project create myproject`.
5.  **Role 할당:** `openstack role add --user myuser --project myproject member`.
6.  **확인:** `myuser`로 로그인해서(`source openrc_myuser`) `myproject`의 자원만 보이는지 확인합니다.

---

## 📝 2주차 과제: Fernet Key Rotation

**목표:** Fernet Token의 보안 메커니즘을 이해하세요.

1.  Keystone 서버(컨테이너)의 `/etc/keystone/fernet-keys/` 디렉토리를 확인하세요.
2.  `keystone-manage fernet_rotate` 명령을 실행하면 키 파일들이 어떻게 변하는지 관찰하세요. (Primary 키가 바뀌고, 오래된 키는 삭제됨)
3.  왜 이 로테이션 작업이 주기적으로(Cron) 필요한지, 클러스터 환경(Multi-node)에서는 이 키 파일을 어떻게 동기화해야 하는지 서술하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** OpenStack에서 자원(VM, Network 등)의 소유권과 격리의 기준이 되는 논리적 단위를 무엇이라 하나요? (Project 또는 Tenant)
2.  **Q2.** Keystone에서 사용하는 최신 토큰 포맷으로, DB에 저장하지 않고 대칭키 암호화를 사용하여 검증하는 방식은? (Fernet Token)
3.  **Q3.** 클라이언트가 "Nova 서비스를 쓰고 싶은데 URL이 뭐야?"라고 알기 위해 조회하는 Keystone의 기능은? (Service Catalog)

---

다음 주, VM을 띄우려면 OS 이미지와 하드디스크가 필요합니다.
이미지 창고 **Glance**와 블록 스토리지 **Cinder**를 배웁니다.

**Authentication is Key.**
