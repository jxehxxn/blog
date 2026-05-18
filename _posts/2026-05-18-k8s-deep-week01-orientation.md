---
layout: post
title: "K8s Deep Week 1: 오리엔테이션 — Kubernetes는 '데이터센터 OS' 다"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series
---

## 학습 목표

- Kubernetes를 한 줄 비유로 머릿속에 박는다.
- 컨트롤 플레인 / 데이터 플레인 구분을 만든다.
- "왜 K8s가 5분이면 컨테이너를 띄우는 마법 도구가 아니라, 복잡한 분산 운영체제인가"를 안다.

## 1. 학부 비유로 시작 — "강의실 1개 OS" vs "캠퍼스 전체 OS"

여러분이 강의실 PC에서 한 학기 동안 사용해 본 **Linux**를 떠올려보세요. Linux는 강의실 PC 한 대의 **CPU 코어 4개, 메모리 16GB, 디스크 500GB**를 여러 프로세스(브라우저, IDE, 컴파일러)에 나눠 줍니다. 누가 CPU를 더 쓸지(스케줄러), 디스크 어디에 쓸지(파일 시스템), 어떤 프로세스가 살아 있는지(프로세스 테이블) 모두 Linux 커널이 관리합니다.

**Kubernetes는 캠퍼스 전체 강의실 100개 PC를 합쳐 운영하는 "캠퍼스 OS"입니다.** 어떤 PC에 어떤 작업을 할당할지(스케줄러), 작업의 데이터는 어디에 둘지(스토리지 추상화), 작업끼리 어떻게 대화할지(네트워크 추상화), 작업이 죽으면 어디서 다시 띄울지(자가복구)를 결정합니다.

이 한 줄을 머리에 박으세요: **Linux는 한 컴퓨터의 OS, Kubernetes는 클러스터의 OS.**

## 2. 컨트롤 플레인과 데이터 플레인 — "두뇌와 손발"

다시 비유: 사람의 몸에는 **두뇌**가 "팔을 들어"라고 결정하고, **손발 근육**이 실제로 움직입니다. 두뇌와 근육은 분리되어 있고 신경(network)을 통해 명령이 전달됩니다.

Kubernetes도 똑같이 분리됩니다.

- **컨트롤 플레인(두뇌)**: 어떤 컨테이너를 어디에 띄울지 결정. etcd / API server / scheduler / controller-manager.
- **데이터 플레인(손발)**: 결정된 명령을 실제로 수행. kubelet / kube-proxy / container runtime.

이 분리가 왜 중요한가? **두뇌가 잠시 잠들어도(컨트롤 플레인 장애) 손발은 이미 받은 명령으로 계속 일합니다.** 즉, master node가 다운돼도 이미 떠 있는 Pod은 계속 동작합니다. 이게 Kubernetes의 "self-healing" 디자인의 핵심 철학입니다.

## 3. "kubectl run nginx" 한 줄에서 일어나는 일 — Full Story

학부생이 가장 헷갈리는 부분입니다. 명령 한 줄 뒤에서 무엇이 일어나는가?

```
[사용자]
  ↓ kubectl run nginx
[kubectl] → REST API 요청 (JSON)
  ↓ HTTPS
[API server]
  ↓ 인증 → 인가(RBAC) → admission webhook
  ↓ "이 Pod 만들어줘" 요청을 etcd에 저장
[etcd] ← Pod 객체 저장 (아직 어디서 돌지 모름)
  ↓ watch event 발생
[scheduler]
  ↓ "어느 노드가 적합한가?" 점수 계산
  ↓ nodeName 결정 → API server로 patch
[etcd] ← Pod에 nodeName=worker-3 기록
  ↓ watch event
[worker-3의 kubelet]
  ↓ "내가 띄워야 할 Pod이 생겼다!"
  ↓ container runtime(containerd)에 컨테이너 생성 요청
[containerd]
  ↓ image pull → cgroup 생성 → 컨테이너 시작
[Pod이 떠서 running]
  ↓ kubelet이 status를 API server에 patch
[etcd] ← status: Running 업데이트
  ↓ watch event
[사용자의 kubectl get pod]
  → Running 표시
```

이 흐름 전체를 외워두세요. 모든 K8s 디버깅의 출발점입니다.

## 4. "선언적(declarative)" — 학생이 가장 헷갈리는 단어

명령형(imperative): "냉장고 문 열어. 우유 꺼내. 잔에 따라." — 절차 지시.
선언적(declarative): "잔에 우유 200ml가 들어 있는 상태가 되어 있으면 좋겠어." — 원하는 결과 지시.

Kubernetes는 후자입니다. 여러분이 `replicas: 3`이라 쓰면 "지금 3개를 만들어"가 아니라 "**언제나 3개가 떠 있는 상태가 되어 있어라**" 라는 의미입니다. 그래서 누군가 Pod 하나를 죽이면 Kubernetes는 새로 띄웁니다 (replicas=3 상태를 유지). 누군가 5개를 직접 띄우면 2개를 죽입니다.

이 모델이 강력한 이유: **항상 같은 결과(idempotent)**. YAML을 100번 apply해도 결과는 동일.

## 5. "왜 그렇게 복잡한가" — 분산 시스템의 8가지 거짓말

L Peter Deutsch가 정리한 "Fallacies of Distributed Computing" 중 일부:
1. 네트워크는 신뢰할 수 있다.
2. 지연 시간은 0이다.
3. 대역폭은 무한하다.
4. 네트워크는 안전하다.
...

이 거짓말을 다 부정하는 시스템이 **분산 시스템**이고, Kubernetes는 본질적으로 분산 OS입니다. 그래서 단순히 "컨테이너 띄우는 도구"가 아니라 etcd(분산 합의), API server(인증/인가), controller(reconciliation loop) 같은 복잡한 컴포넌트가 모두 필요합니다.

학부생을 위한 핵심 인사이트: **Kubernetes의 복잡함은 분산 시스템의 본질적 어려움을 반영한 것이지, 일부러 만든 것이 아니다.**

## 6. 한 학기 동안 우리가 다룰 컴포넌트 미리보기

```
[컨트롤 플레인]
  - etcd: 캠퍼스 OS의 "기억장치". 모든 객체의 진실의 원천.
  - API server: 모든 명령이 통과하는 "교문".
  - scheduler: "어느 강의실에 어느 학생을?" 결정.
  - controller-manager: "현실이 계획과 다르면 고친다" — 자가복구 두뇌.

[데이터 플레인 — 각 노드마다]
  - kubelet: "이 강의실 담당 조교". Pod을 실제로 띄움.
  - kube-proxy: "강의실 사이 우편함". Service IP 트래픽 라우팅.
  - container runtime (containerd/CRI-O): 컨테이너를 직접 실행.

[애드온]
  - CNI: Pod 간 네트워크 (4주차).
  - CSI: 영구 스토리지 (6주차).
  - CoreDNS: 클러스터 안의 "전화번호부".
  - metrics-server: 자원 사용량 측정.
```

## 7. 실습 — kind로 1분 만에 클러스터 띄우기

```bash
# Mac/Linux
brew install kind kubectl

cat <<EOF | kind create cluster --name learn --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

kubectl get nodes
# 1 control-plane + 2 worker

# 컨트롤 플레인 컴포넌트 직접 보기
kubectl get pods -n kube-system
```

`kube-system` namespace에 어떤 Pod이 있는지 한 줄씩 확인하세요. 위 4 컴포넌트가 모두 보입니다.

## 8. 자가평가 퀴즈

### Q1. Kubernetes를 한 줄로?
1. 컨테이너 빌드 도구
2. **클러스터(여러 노드)를 한 OS로 다루는 분산 OS**
3. CI/CD 시스템
4. 모니터링 도구

**정답: 2.**

### Q2. 컨트롤 플레인이 잠시 다운되면?
1. 모든 Pod이 즉시 죽음
2. **이미 떠 있는 Pod은 계속 동작, 새 Pod 스케줄링만 멈춤**
3. 네트워크가 끊김
4. 데이터 손실

**정답: 2.** 두뇌-손발 비유의 핵심.

### Q3. "선언적" 모델의 핵심 가치는?
1. 빠른 빌드
2. **idempotent — 100번 apply해도 결과 동일, drift 자동 복구**
3. UI 미려
4. 무관

**정답: 2.**

### Q4. `kubectl run nginx` 후 가장 먼저 객체가 저장되는 곳은?
1. kubelet
2. scheduler
3. **etcd**
4. containerd

**정답: 3.** scheduler는 etcd에 저장된 객체를 watch 해서 nodeName을 결정.

### Q5. Kubernetes의 복잡함이 본질적인 이유는?
1. 일부러 어렵게 만들어서
2. **분산 시스템의 본질적 어려움(네트워크 거짓말 등)을 반영해야 해서**
3. Go로 짜서
4. 무관

**정답: 2.**

## 9. 다음 주차

[Week 2: 컨트롤 플레인 깊게]에서는 etcd(분산 KV 합의), API server(인증/인가/admission), scheduler(점수 계산), controller-manager(reconciliation)를 한 줄 한 줄 풀어봅니다.
