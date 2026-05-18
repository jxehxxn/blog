---
layout: post
title: "K8s Deep Week 2: 컨트롤 플레인 깊게 — etcd, API server, scheduler, controller-manager"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series
---

## 학습 목표

- etcd의 Raft 합의 알고리즘을 비유로 이해한다.
- API server의 4단계 처리(auth → authz → admission → storage)를 안다.
- Scheduler의 두 단계(filter → score)를 손으로 따라간다.
- Controller-manager의 reconciliation loop를 코드 한 페이지로 본다.

## 1. etcd — "캠퍼스 OS의 기억"

비유: 학생회실 게시판이 4개 캠퍼스에 분산되어 있다고 합시다. 누가 게시판에 글을 새로 붙이면 4개 게시판 중 **과반(3개 이상)**이 같은 내용을 적은 후에만 "확정"입니다. 그래서 게시판 1개가 불에 타도 나머지 3개로 진실을 복원할 수 있습니다.

이게 **Raft 합의 알고리즘**입니다. etcd는 이걸로 동작합니다.

기술적으로:
- etcd 노드는 보통 3 또는 5개 (홀수).
- 한 노드가 leader, 나머지는 follower.
- 쓰기 요청은 leader가 받아 follower에 복제, 과반 ACK 후 commit.
- leader가 죽으면 follower가 새 leader 선출.

K8s에서 etcd가 멈추면 어떤 일이? **모든 신규 변경이 멈춥니다.** 이미 떠 있는 Pod은 무관하지만, kubectl apply는 hang. 그래서 etcd 가용성은 컨트롤 플레인의 생명선입니다.

`etcdctl`로 직접 보기:

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get / --prefix --keys-only | head -20
```

`/registry/pods/default/nginx` 같은 키가 보입니다. **K8s의 모든 객체는 etcd에 이렇게 키-값으로 저장**됩니다.

## 2. API Server — "교문"

비유: 캠퍼스 정문 경비실에서 모든 출입자가 (1) 신분 확인, (2) 출입 권한 확인, (3) 가방 검사를 거치고 통과합니다. 통과한 정보는 학적 데이터베이스에 기록됩니다.

API server의 4단계:

```
요청 → [Authentication] → [Authorization] → [Admission] → [etcd 저장]
        누구인가?         허락됐는가?         규칙위반은?    영구 기록
```

### (1) Authentication
- 인증서, JWT, ServiceAccount 토큰 등으로 "이 요청자가 누구"인지 확인.
- 실패하면 401.

### (2) Authorization (RBAC)
- "이 사용자가 이 리소스에 이 동작을 할 권한이 있는가?"
- RoleBinding/ClusterRoleBinding으로 정의.
- 실패하면 403.

### (3) Admission Controller — 학부생을 위한 비유
가방 검사에 비유합니다. 두 종류 검사관이 있습니다.
- **Validating admission**: "가방 안에 칼이 들어 있나?" 검사 후 통과/거부만.
- **Mutating admission**: "물병에 라벨이 없네, 내가 라벨 붙여줄게" — 요청 자체를 수정.

대표적 admission:
- `NamespaceLifecycle`: 존재하지 않는 namespace에 리소스 생성 거부.
- `LimitRanger`: 리소스 limit 자동 추가.
- `PodSecurity` (PSA): privileged pod 거부.
- Custom webhook: OPA Gatekeeper, Kyverno (Tier 1 코스 6번에서).

### (4) etcd 저장
앞 단계 모두 통과하면 etcd에 객체 저장. 다른 컴포넌트(scheduler, controller)가 watch로 변화 감지.

## 3. Scheduler — "어느 강의실?"

비유: 200명 신청한 수업의 강의실 배정 담당자. (1) 자리가 부족한 강의실은 후보에서 제외, (2) 남은 후보에 점수를 매겨 (장비, 위치, 사용 빈도) 가장 좋은 곳 선택.

기술적으로 **2단계**입니다.

### (1) Filtering (Predicates)
조건을 만족하지 않는 노드 제외.
- `PodFitsResources`: CPU/메모리 충분?
- `NoDiskConflict`: 볼륨 충돌 없음?
- `MatchNodeSelector`: nodeSelector 매칭?
- `PodToleratesNodeTaints`: taint 허용?

### (2) Scoring (Priorities)
남은 노드에 점수 (0~10) 부여 후 합산.
- `LeastRequestedPriority`: 가용 자원 많은 노드 우선.
- `BalancedResourceAllocation`: CPU/메모리 비율 균형.
- `NodeAffinityPriority`: affinity 매칭.
- `ImageLocalityPriority`: 이미지 이미 있는 노드 우선 (image pull 시간 절감).

가장 높은 점수의 노드를 선택해 `pod.spec.nodeName = "worker-3"` 으로 patch. 이후 worker-3의 kubelet이 Pod을 실제로 띄움.

자세한 scoring 알고리즘은 7주차에서 다룹니다.

## 4. Controller Manager — "현실과 계획의 차이를 메꾼다"

비유: 강의실 관리자가 "이 강의실엔 의자가 30개 있어야 한다"는 표를 보고, 실제 의자가 25개면 5개 더 채우고, 35개면 5개를 치웁니다. **표(desired)와 현실(actual)의 차이를 영원히 메꾸는 사람.**

이게 **reconciliation loop**입니다. controller-manager 안에는 30+ 개 controller가 들어 있고, 각각 자기 리소스를 책임집니다.

### 핵심 controller 몇 개

| Controller | 책임 | 비유 |
|------------|------|------|
| Deployment | ReplicaSet 관리 | 학사관리자: "롤아웃 정책에 맞춰 새 학기 강의 명단 갱신" |
| ReplicaSet | Pod 개수 유지 | 강의실 관리자: 의자 N개 유지 |
| Node | 노드 상태 추적 | 시설관리자: 강의실 작동 여부 |
| Endpoint | Service ↔ Pod 매핑 | 교내 전화번호부 갱신 |
| Job | 일회성 작업 완료 추적 | 시험 감독 |
| CronJob | 주기 작업 | 시간표 알람 |

### Reconciliation 코드 패턴

```go
for {
    desired := getDesiredState()       // etcd에서 표 읽기
    actual  := getActualState()        // 실제 상태 조회
    diff    := compare(desired, actual)
    if diff != nil {
        applyChanges(diff)             // 차이 보정
    }
    waitForNextEvent()                 // 변화 알림 기다림
}
```

이 패턴이 K8s 전체를 떠받칩니다. Operator 개발(8주차)도 같은 패턴 그대로 재사용.

## 5. 컴포넌트 사이 통신 — Watch & List

학부생이 자주 묻는 질문: "controller가 어떻게 변화를 알지? 매번 polling 하나?"

답: **Watch**. HTTP long-poll로 API server에 연결해두면, 변화 발생 시 API server가 controller에게 push. polling이 아니라 reactive.

```
controller ──[long HTTP GET ?watch=true]──→ API server
                                              │
                                              ↓
                                            etcd (변화 발생)
                                              │
                                              ↓ watch event
controller ←─[chunked HTTP response]─────── API server
```

만약 watch 연결이 끊기면 controller는 (1) 전체 list 다시 받고 (2) resourceVersion부터 다시 watch.

## 6. 실습

```bash
# 1. etcd 직접 조회
kubectl exec -n kube-system etcd-kind-control-plane -- \
  etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/pods --prefix --keys-only | head

# 2. API server audit log 켜기 (이미 켜진 cluster면 보기)
kubectl get --raw /metrics | grep apiserver_request_total | head

# 3. scheduler 의사결정 로그
kubectl -n kube-system logs deploy/kube-scheduler | head -30
# (kind 클러스터에서는 kube-scheduler가 static pod)

# 4. controller-manager 로그
kubectl -n kube-system logs <kube-controller-manager-pod> | head
```

각 컴포넌트의 로그 한 페이지를 직접 보며 "내가 배운 흐름과 어떻게 매칭되는지" 손으로 따져보세요.

## 7. 자가평가 퀴즈

### Q1. etcd가 홀수 노드인 이유는?
1. 짝수면 동작 안 함
2. **과반(quorum) 계산이 명확해서 — 3 중 2, 5 중 3**
3. 비용 절감
4. 무관

**정답: 2.**

### Q2. API server의 admission controller 중 mutating 의 역할은?
1. 요청을 거부만
2. **요청 자체를 수정 (라벨 추가, 사이드카 주입 등)**
3. 인증
4. 무관

**정답: 2.**

### Q3. Scheduler의 2단계 중 1단계는?
1. **Filtering — 조건 안 맞는 노드 제외**
2. Scoring
3. Eviction
4. Binding

**정답: 1.**

### Q4. Controller-manager의 핵심 패턴은?
1. **Reconciliation — desired와 actual 차이를 영원히 메꿈**
2. Polling
3. Push notification
4. Batch job

**정답: 1.**

### Q5. Controller가 변화를 감지하는 방식은?
1. Polling
2. **Watch (long HTTP)**
3. Webhook
4. cron

**정답: 2.**

## 8. 다음 주차

[Week 3: 데이터 플레인]에서는 노드 위의 kubelet, kube-proxy, container runtime을 다룹니다. 컨테이너가 "실제로 어떻게" 떠 있는지(cgroups, namespaces, OCI spec) 풀어봅니다.
