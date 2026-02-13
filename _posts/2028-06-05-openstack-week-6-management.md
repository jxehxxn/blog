---
layout: post
title: "OpenStack Zero-to-Hero: Week 6 - Management Interfaces (Horizon, CLI, SDK)"
---

오픈스택을 제어하는 방법은 3가지가 있습니다.
1.  **Horizon (Web Dashboard):** 마우스 클릭으로 VM 생성. 초보자용.
2.  **CLI (OpenStack Client):** 터미널에서 스크립트 실행. 중급자/운영자용.
3.  **SDK (Python):** 파이썬 코드로 제어. 자동화/개발자용.

전문가는 CLI를 주력으로 쓰고, 자동화할 때 SDK를 씁니다. Horizon은 현황 파악용입니다.

---

## 1. OpenStack Client (CLI)

예전에는 `nova list`, `neutron net-list` 처럼 명령어가 따로 놀았지만, 지금은 `openstack` 명령어 하나로 통합되었습니다.

### Authentication (`openrc`)
CLI를 쓰려면 환경 변수를 로드해야 합니다.
```bash
export OS_AUTH_URL=http://controller:5000/v3
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=secret
```
이걸 파일(`admin-openrc`)로 만들고 `source admin-openrc` 하면 됩니다.

### Output Format
스크립트 짤 때 유용한 옵션입니다.
*   `openstack server list -f json`: JSON 출력 (jq 파싱용)
*   `openstack server list -f value -c ID`: ID만 쭉 출력.

---

## 2. Python SDK (`openstacksdk`)

REST API를 직접 호출하는 건 너무 힘듭니다. SDK를 쓰세요.

```python
import openstack

conn = openstack.connect(cloud='mycloud')

# VM 목록 조회
for server in conn.compute.servers():
    print(server.name, server.id)

# VM 생성
conn.compute.create_server(
    name="my-vm", 
    flavor_id="m1.small", 
    image_id="ubuntu-22.04"
)
```
`clouds.yaml` 파일에 접속 정보를 넣어두면 코드가 깔끔해집니다.

---

## 3. Horizon Customization

Horizon은 Django 기반 웹 앱입니다.
회사 로고를 박거나, 메뉴를 숨기거나(Policy), 테마를 바꿀 수 있습니다.
`/etc/openstack-dashboard/local_settings.py`를 수정하면 됩니다.

---

## 🛠️ Lab: Automating VM Creation

CLI 스크립트로 VM 10대를 한 방에 만들어봅니다.

```bash
#!/bin/bash
IMAGE_ID=$(openstack image show cirros -c id -f value)
NET_ID=$(openstack network show private -c id -f value)

for i in {1..10}; do
  openstack server create --image $IMAGE_ID --flavor m1.tiny --network $NET_ID web-$i &
done
wait
echo "All VMs created!"
```

---

## 📝 6주차 과제: SDK Scripting

**목표:** Python SDK를 사용하여 **"일주일 이상 된 VM을 찾아서 종료(Stop)"**하는 관리 스크립트를 작성하세요.

1.  `conn.compute.servers()`로 모든 서버를 가져옵니다.
2.  `server.created_at` 속성을 파싱하여 현재 시간과 비교합니다. (`datetime` 모듈 사용)
3.  7일이 지났고 상태가 `ACTIVE`라면 `conn.compute.stop_server(server)`를 호출합니다.
4.  건드린 VM 목록을 로그로 남깁니다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** OpenStack CLI 명령어를 사용할 때 인증 정보를 환경 변수로 로드하기 위해 실행하는 쉘 스크립트 파일의 일반적인 이름은? (openrc 또는 admin-openrc)
2.  **Q2.** OpenStack Python SDK를 사용할 때, 클라우드 접속 정보(Auth URL, User, Password 등)를 저장하는 표준 YAML 파일의 이름은? (clouds.yaml)
3.  **Q3.** Horizon 대시보드에서 특정 기능(메뉴)을 일반 사용자에게 보이지 않게 숨기려면 어떤 파일을 수정해야 하나요? (local_settings.py 또는 정책 파일 policy.yaml)

---

다음 주, 이제 수동 설치는 그만.
Docker 컨테이너 기반으로 오픈스택을 배포하는 **Kolla-Ansible**을 배웁니다.
이걸 알면 설치 시간이 3일에서 30분으로 줄어듭니다.

**CLI Mastered.**
