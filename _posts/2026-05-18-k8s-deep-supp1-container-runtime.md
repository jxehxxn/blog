---
layout: post
title: "K8s Deep 보충 1: 컨테이너 런타임 — containerd vs CRI-O, Docker의 죽음"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series supplement
---

3주차 데이터 플레인에서 짧게 다룬 container runtime을 깊게 정리합니다.

## 1. 비유 — "주방의 셰프"

kubelet이 "이 요리(컨테이너) 만들어줘" 라고 부탁하는 셰프가 runtime. 각 셰프는 같은 표준 레시피(OCI)를 따르지만 주방 도구·일하는 방식이 다릅니다.

## 2. 표준 — OCI

OCI(Open Container Initiative)가 두 가지 표준을 정의:
- **OCI Runtime Spec**: 컨테이너를 어떻게 실행하는가 (namespace/cgroup 설정 등).
- **OCI Image Spec**: 컨테이너 이미지가 어떻게 생겼는가 (manifest, layer).

거의 모든 runtime이 이 표준을 따름.

## 3. 계층 구조

```
[kubelet]
   ↓ CRI (gRPC)
[high-level runtime: containerd / CRI-O]
   ↓ OCI runtime spec
[low-level runtime: runc / crun / kata]
   ↓ Linux syscalls
[OS Kernel — namespace, cgroup, ...]
```

- High-level runtime: 이미지 관리, CRI 변환, snapshot, network plugin 호출.
- Low-level runtime: namespace/cgroup 직접 만들고 프로세스 fork+exec.

## 4. Docker의 죽음 — Dockershim Deprecation

K8s v1.20 deprecated, v1.24 완전 제거. 이유:

- Docker는 K8s 시작 전부터 존재. CRI가 없어서 K8s가 "dockershim" 어댑터 직접 유지.
- containerd가 표준화되며 dockershim은 중복 코드.
- maintenance burden 떨치고 표준 CRI만 지원하기로.

**Docker 자체는 죽지 않음.** Docker Desktop, dev환경, docker build는 여전히 강력. 단 K8s production cluster의 runtime으로는 더 이상 사용 불가.

## 5. containerd

- CNCF graduated, 가장 널리 쓰임.
- AWS EKS, GKE, AKS 기본.
- Docker가 사용하는 runtime 부분을 따로 떼낸 것 → 사실상 Docker와 같은 코어.
- 이미지 관리, snapshot, container lifecycle 책임.

설정:
```bash
crictl ps                                # CRI client
crictl images
ctr -n k8s.io containers list             # containerd 직접
```

## 6. CRI-O

- Red Hat 주도, OpenShift 기본.
- K8s만을 위해 최소화된 runtime.
- 기능 단순 → 보안 표면 작음.
- 빅테크 중 Red Hat 진영 + 일부 보안 강화 환경에서 선호.

## 7. runc / crun / Kata / gVisor

low-level runtime 비교:

| Runtime | 특징 | 사용 |
|---------|------|------|
| runc | 표준, OCI reference | 기본 |
| crun | C 구현, 빠르고 가벼움 | 일부 환경 |
| Kata Containers | VM 기반 강한 격리 | regulated workload |
| gVisor | userspace kernel | Google GKE Sandbox |

Kata/gVisor는 VM 또는 user-space kernel로 격리 강화. 보안 critical 워크로드 (멀티테넌트 SaaS, untrusted code) 에 사용.

## 8. 빌드와 런타임의 분리

K8s가 runtime을 쓰는 건 "실행"이고, **이미지 빌드는 별개**. Docker 없이 빌드:
- **kaniko**: 컨테이너 안에서 daemon 없이 빌드.
- **buildah**: Red Hat의 빌드 도구.
- **img**, **BuildKit**.

CI/CD pipeline에서 daemonless 빌드가 표준이 되어 가는 중.

## 9. 운영 함정

1. Docker만 알고 K8s 들어가서 dockershim deprecation에 당황.
2. crictl 명령 모르고 디버깅 어려움.
3. Kata/gVisor 도입 시 성능 5~30% 손실 고려.
4. containerd 버전 ~ K8s 호환표 무시.
5. /var/lib/containerd 디스크 full → 노드 장애.

## 10. 학부 실습

```bash
# 1. containerd가 띄운 컨테이너 직접 조회
sudo crictl ps

# 2. 이미지 풀
sudo crictl pull nginx:latest

# 3. ctr로 더 낮은 레벨 보기
sudo ctr -n k8s.io images list

# 4. runc로 직접 컨테이너 실행 (학습)
mkdir mycont && cd mycont
runc spec
runc run mycont
```

## 11. 결론

"컨테이너 = Docker" 등식은 K8s 세계에서 깨졌습니다. containerd가 사실상 표준. 다만 dev 환경에서 Docker는 여전히 편리. 시니어는 두 계층(high/low runtime)을 모두 이해해야 디버깅이 됩니다.
