---
layout: post
title: "Kubernetes 깊이 알기 (Senior Platform Series 1/19): 12주 강의 계획서"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series
---

## 이 강의가 누구를 위한 것인가

학부생 ~ 주니어 엔지니어가 **빅테크 Senior Platform Engineer** 수준의 Kubernetes 이해에 도달하기 위한 12주 과정입니다.

전제 비유: **Kubernetes는 데이터센터 운영체제다.** Linux가 한 컴퓨터의 CPU·메모리·디스크를 프로세스에 분배하듯, Kubernetes는 클러스터의 CPU·메모리·네트워크·스토리지를 컨테이너에 분배합니다. 이 강의는 그 OS의 커널을 해부합니다.

## 사전 지식

- Linux 셸 (필수)
- Docker 기초 (필수)
- TCP/IP, HTTP 기초 (필수)
- Go 기초 (8주차 Operator 작성)
- `kubectl apply` 정도는 해본 경험 (필수)

## 학습 결과

12주 후 다음을 할 수 있어야 합니다.

1. 사용자가 `kubectl run nginx` 한 순간부터 Pod이 컨테이너로 뜨기까지 모든 컴포넌트의 흐름을 그릴 수 있다.
2. CNI 플러그인 3종(Calico/Cilium/Flannel)의 동작 원리를 비교할 수 있다.
3. CSI 드라이버를 직접 따라 읽고, 신규 스토리지 백엔드 연결 절차를 설명할 수 있다.
4. 커스텀 스케줄러를 작성할 수 있다.
5. CRD + Controller(Operator)를 kubebuilder로 처음부터 작성할 수 있다.
6. 10,000 노드 클러스터의 성능 병목을 진단할 수 있다.

## 주차 계획

| 주 | 주제 | 형식 |
|----|------|------|
| 1 | 오리엔테이션 — K8s가 "OS" 인 이유 | 강의 |
| 2 | 컨트롤 플레인 — etcd, API server, scheduler, controller-manager | 강의+실습 |
| 3 | 데이터 플레인 — kubelet, kube-proxy, container runtime | 강의+실습 |
| 4 | Networking — CNI, IPAM, pod-to-pod 패킷 추적 | 실습 |
| 5 | Service / Ingress / Gateway API | 실습 |
| 6 | Storage — PV/PVC/StorageClass/CSI | 실습 |
| 7 | Scheduler 깊게 — predicates, priorities, custom scheduler | 실습 |
| 8 | Operator 개발 — kubebuilder로 CRD+Controller | 실습 |
| 9 | 보안 — RBAC, PSA, NetworkPolicy, etcd encryption | 강의+실습 |
| 10 | Performance & Scale — etcd tuning, API priority/fairness | 강의 |
| 11 | Multi-tenancy — namespace, hierarchical NS, vCluster | 강의 |
| 12 | 캡스톤 — 커스텀 Operator + multi-tenant 클러스터 | 프로젝트 |

## 보충 포스트 (Supplements)

본 강의에서 "심화는 보충" 이라 짚은 모든 부분을 다음 보충 포스트가 채웁니다.

- 보충 1: 컨테이너 런타임 비교 — containerd / CRI-O / Docker 의 죽음
- 보충 2: CNI 3종 비교 — Calico / Cilium / Flannel
- 보충 3: CSI 깊게 — block / file / object storage
- 보충 4: etcd 운영 — backup, restore, defrag, 멀티노드

## 평가

- 매 주차 자가평가 (5문항): 30%
- 주차별 실습 과제: 30%
- 캡스톤(루브릭+모범답안 3종): 40%

## 다음 주차

[Week 1: 오리엔테이션]에서는 Kubernetes를 "데이터센터 OS" 라는 한 문장으로 정리하고, Linux 비유로 모든 컴포넌트의 역할을 미리 머릿속에 그려봅니다.
