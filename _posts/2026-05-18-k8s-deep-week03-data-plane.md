---
layout: post
title: "K8s Deep Week 3: 데이터 플레인 — kubelet, kube-proxy, container runtime"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series
---

## 학습 목표

- kubelet이 PodSpec을 받아 실제 컨테이너로 만드는 흐름을 안다.
- container runtime(containerd / CRI-O)의 역할과 OCI 표준을 안다.
- "컨테이너"의 정체 — namespaces + cgroups를 손으로 직접 확인한다.
- kube-proxy의 iptables/IPVS 모드를 비유로 이해한다.

## 1. kubelet — "강의실 조교"

비유: 캠퍼스 OS(컨트롤 플레인)는 "worker-3 강의실에서 nginx 수업 진행해" 라고 결정합니다. 그러면 worker-3 강의실의 **조교(kubelet)**가 이 명령을 받고 (1) 강의 자료를 준비(image pull), (2) 강의실 자원을 세팅(cgroups), (3) 출석을 추적(liveness probe), (4) 강의 종료 후 청소(cleanup) 합니다.

기술적으로 kubelet은:
- 각 노드에 데몬으로 실행.
- API server의 `pods` endpoint를 watch — 자기 노드에 할당된 Pod 발견 시 작업.
- container runtime에 OCI 명세대로 컨테이너 요청.
- liveness/readiness probe 주기적 실행.
- Pod의 status를 주기적으로 API server에 보고.

### kubelet과 PodSpec

`pod.spec.containers[0].image = nginx:1.21` 을 kubelet이 받으면:
1. 이미지가 노드에 이미 있나? 없으면 registry에서 pull.
2. cgroup namespace 생성.
3. container runtime에 "이 이미지로 컨테이너 실행" 요청.
4. 컨테이너 PID 받아서 추적.

## 2. Container Runtime — "실제 강의 진행자"

비유: 조교(kubelet)가 "강의 시작해" 라고 부탁하면, 실제 강의실 앞에 서서 강의를 진행하는 사람이 **runtime(containerd)** 입니다. 강의 자료(image)를 받아 칠판에 띄우고(filesystem mount), 학생들의 자리(memory)를 정해주고(cgroups), 다른 강의(다른 컨테이너)와 섞이지 않게 분리(namespaces) 합니다.

기술적으로:
- containerd / CRI-O: OCI 호환 runtime.
- kubelet이 **CRI(Container Runtime Interface)**라는 gRPC API로 runtime과 통신.
- runtime은 다시 **runc** 같은 low-level runtime을 호출해 실제 Linux namespace/cgroup 만듦.

```
[kubelet] ─[CRI gRPC]→ [containerd] ─[OCI shim]→ [runc] → [Linux namespace + cgroup]
                                                            └─→ 컨테이너 프로세스
```

자세한 비교(containerd vs CRI-O vs Docker)는 [보충 1: 컨테이너 런타임 비교] 포스트.

## 3. 컨테이너의 정체 — namespaces + cgroups

학부생이 가장 헷갈리는 질문: **"컨테이너는 진짜 가상머신인가? 어떻게 격리되나?"**

답: **컨테이너는 가상머신이 아니라, 한 Linux 호스트의 일반 프로세스에 강력한 격리(namespaces) + 자원 제한(cgroups) 옷을 입힌 것.**

### Linux Namespaces (7종)

비유: 도서관 칸막이가 7종류 있어서 (1)자기 책상만 보이고 (2)다른 사람 책상에는 발이 못 닿게 합니다.

| Namespace | 격리하는 것 |
|-----------|-------------|
| pid | 프로세스 ID — 컨테이너 안에서는 PID 1이 자기 자신 |
| net | 네트워크 인터페이스, 라우팅, iptables |
| mount | 파일시스템 마운트 포인트 |
| uts | 호스트네임 |
| ipc | 공유 메모리, 메시지 큐 |
| user | UID/GID 매핑 |
| cgroup | cgroup 트리 |

`lsns` 명령으로 직접 보기:

```bash
sudo lsns
#  NS TYPE   NPROCS PID USER COMMAND
4026531836 pid    150   1 root /sbin/init
4026532225 pid      1 23456 1000 sleep 1000  ← 컨테이너 안의 sleep, PID 1로 보임
```

### Cgroups — "자원 가위"

비유: 강의실에 학생 100명을 넣어도 "이 그룹 30명은 칠판 앞쪽만 써, CPU 1코어만 써" 같은 제한을 거는 가위.

```bash
# 컨테이너가 사용하는 CPU/메모리 제한 보기
cat /sys/fs/cgroup/cpu.max
# 100000 100000  ← 1 CPU 사용 가능
cat /sys/fs/cgroup/memory.max
# 134217728     ← 128 MB
```

`pod.spec.containers[0].resources.limits.memory = "128Mi"` 가 결국 cgroup의 memory.max로 떨어집니다.

### 직접 격리 만들기 (학부 실습)

```bash
# Linux에서 직접 namespace + cgroup으로 "미니 컨테이너" 만들기
sudo unshare --pid --net --mount --uts --ipc --fork bash
# 새 namespace 안의 셸
hostname mini
ps -ef
# PID 1이 bash로 보임 ← 격리됨
```

Docker/K8s 없이도 컨테이너 본질을 5분에 체험.

## 4. kube-proxy — "강의실 사이 우편함"

비유: 학생이 "교수 A님 강의실" 을 찾으면 항상 정해진 번호의 우편함(Service IP)으로 편지를 넣습니다. 우편함은 매번 다른 강의실(Pod)로 편지를 전달합니다. 강의실이 바뀌어도 학생은 우편함 번호만 알면 됩니다.

이게 **Service**의 비유고, kube-proxy가 우편 배달부입니다.

### kube-proxy의 3가지 모드

1. **userspace** (deprecated): kube-proxy가 직접 패킷 받아 forward. 느림.
2. **iptables**: Linux iptables NAT 규칙을 kube-proxy가 자동 생성. 패킷은 커널이 처리 (빠름).
3. **IPVS**: Linux IPVS(L4 LB) 사용. iptables보다 빠르고 다양한 로드밸런싱 알고리즘.

### iptables 모드 동작

```bash
# Service ClusterIP = 10.96.0.10
# Pod IP = 10.244.0.5, 10.244.0.6, 10.244.0.7

iptables -t nat -L KUBE-SERVICES
# KUBE-SVC-XXXX (10.96.0.10) → 33% 확률 SEP-A, 33% SEP-B, 33% SEP-C

iptables -t nat -L KUBE-SEP-A
# DNAT 10.244.0.5
```

학생 입장: Service IP로 패킷 → iptables NAT가 무작위 Pod IP로 destination 변환. 마치 우편함이 무작위 강의실로 전달.

빅테크에서는 수천 Service에서 iptables 규칙이 폭증해 성능 저하 → **IPVS 모드 권장**.

## 5. Pause Container — 학부생이 모르는 비밀

`kubectl describe pod` 하면 항상 보이는 정체 모를 "pause" 컨테이너. 이건 뭔가?

비유: 강의실 하나에 여러 강사(컨테이너)가 들어와 같은 칠판을 공유하려면, 누군가가 먼저 강의실 문을 열고 자리를 잡고 있어야 합니다. 그 "자리잡기" 역할이 pause 컨테이너.

기술적으로:
- Pod 안의 모든 컨테이너가 **같은 network namespace + IPC namespace**를 공유.
- pause 컨테이너가 먼저 namespace를 만들고 잠자며 holding.
- 다른 컨테이너들이 그 namespace에 join.

그래서 같은 Pod의 두 컨테이너는 `localhost`로 서로 통신 가능.

## 6. 실습 — 컨테이너 직접 해부

```bash
# 1. kind 클러스터 띄움 (지난 주)
# 2. nginx Pod 띄움
kubectl run nginx --image=nginx
kubectl get pod nginx -o wide
# nginx 1/1 Running ... 10.244.0.5 worker-3

# 3. worker-3 노드 안으로 들어감 (kind에서는 docker exec)
docker exec -it learn-worker bash

# 4. containerd로 컨테이너 직접 조회
crictl ps | grep nginx

# 5. 컨테이너의 PID로 namespace 보기
PID=$(crictl inspect $(crictl ps -q --name=nginx) | jq -r '.info.pid')
ls -l /proc/$PID/ns/
# pid, net, mnt 등 namespace 파일들

# 6. cgroup 보기
cat /sys/fs/cgroup/$(cat /proc/$PID/cgroup | head -1 | cut -d: -f3)/memory.max
```

이 흐름을 직접 해보면 "컨테이너 = namespace + cgroup" 이 머리에 박힙니다.

## 7. 자가평가 퀴즈

### Q1. kubelet의 책임 중 가장 큰 것은?
1. **노드에 할당된 Pod을 실제 컨테이너로 띄우고 상태 보고**
2. 스케줄링
3. 인증
4. UI

**정답: 1.**

### Q2. CRI는 무엇의 약자?
1. **Container Runtime Interface — kubelet과 runtime의 gRPC 표준**
2. Container Reduced Image
3. CRI는 약자 아님
4. Cluster Resource Interface

**정답: 1.**

### Q3. "컨테이너"의 본질은?
1. 작은 VM
2. **Linux namespace + cgroup으로 격리·제한된 일반 프로세스**
3. Docker 전용 기술
4. JVM

**정답: 2.**

### Q4. Pod 안 두 컨테이너가 localhost 통신이 가능한 이유는?
1. **같은 network namespace 공유 (pause 컨테이너 덕분)**
2. 자동 NAT
3. iptables 마법
4. 무관

**정답: 1.**

### Q5. kube-proxy의 IPVS 모드가 iptables보다 권장되는 이유?
1. UI 표시
2. **수천 Service에서 성능·확장성 우수**
3. 무관
4. 비용 절감

**정답: 2.**

## 8. 다음 주차

[Week 4: Networking — CNI]에서는 Pod IP가 어디서 나오는지, pod-to-pod 패킷이 노드를 어떻게 건너가는지, Calico/Cilium 차이는 무엇인지 다룹니다.
