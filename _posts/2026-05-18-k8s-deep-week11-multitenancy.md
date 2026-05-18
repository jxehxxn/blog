---
layout: post
title: "K8s Deep Week 11: Multi-tenancy — Namespace, Hierarchical NS, vCluster"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series multi-tenancy
---

## 학습 목표

- Multi-tenancy의 3가지 등급(soft/hard/virtual)을 안다.
- Namespace + RBAC + Quota로 soft multi-tenancy를 구현한다.
- Hierarchical Namespace Controller(HNC)의 가치를 안다.
- vCluster로 cluster-in-cluster 격리를 한다.

## 1. 비유 — "공동주택의 3 종류"

- **Soft**: 같은 아파트, 다른 호수 (Namespace 격리).
- **Hard**: 같은 단지 다른 동, 다른 입구·관리비 (cluster 분리).
- **Virtual**: 같은 동에 가짜 호수 가벽 (vCluster — control plane 분리).

빅테크는 보통 셋 다 혼합. 팀 격리는 soft, 규제 영역(PCI/HIPAA)은 hard, 개발용 임시 환경은 virtual.

## 2. Soft Multi-tenancy

같은 클러스터 안에서 namespace로 분리.

### 핵심 컴포넌트
- **Namespace**: 객체 grouping.
- **RBAC**: 사용자/SA 별 권한 격리.
- **ResourceQuota**: namespace 자원 한도.
- **LimitRange**: Pod 기본 limit.
- **NetworkPolicy**: 트래픽 격리.
- **PSA**: privileged Pod 차단.

### ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payments-quota
  namespace: payments
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    persistentvolumeclaims: "20"
    count/deployments.apps: "50"
```

namespace 안 모든 자원 합계가 quota 초과 시 새 Pod 생성 거부.

### LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata: { name: defaults, namespace: payments }
spec:
  limits:
    - type: Container
      default: { cpu: 500m, memory: 256Mi }
      defaultRequest: { cpu: 100m, memory: 128Mi }
      max: { cpu: "4", memory: 8Gi }
```

limit/request 명시 안 한 Pod에 자동 적용. 한 Pod이 너무 큰 limit 못 잡도록.

## 3. Soft의 한계

같은 클러스터의 root-equivalent 컴포넌트(kubelet, container runtime, kernel)를 공유 → kernel exploit 한 번이면 격리 깨짐.

빅테크 정책: PCI/HIPAA/금융 데이터는 soft로 부족. cluster 분리(hard) 또는 노드 분리(taint 기반).

## 4. Hierarchical Namespace Controller (HNC)

비유: namespace에 부모-자식 관계를 줘서 정책 자동 상속. "team-payments" 밑에 "payments-app1", "payments-app2" namespace를 두면 부모의 RBAC/Quota가 자동 상속.

```bash
kubectl hns create payments-app1 -n payments
```

부모 namespace의 RoleBinding이 자식에 자동 복제. quota도 부모-자식 합산 가능.

빅테크 자체 platform 위에 HNC 비슷한 추상화 직접 만든 사례 많음.

## 5. vCluster — Virtual Cluster

비유: 같은 한 K8s 클러스터 안에 "가짜 K8s 클러스터"를 띄움. 가짜 클러스터의 API server / etcd / scheduler가 namespace 안에 들어 있고, 실제 Pod은 호스트 클러스터의 노드에 sync.

```bash
vcluster create my-vcluster --namespace my-vcluster
vcluster connect my-vcluster
kubectl get nodes   # 가짜 노드 보임
```

장점:
- 사용자는 cluster-admin 수준 권한 가짐 (가짜 클러스터에 한해).
- 호스트 클러스터는 영향 없음.
- 빠른 dev/test 환경.

단점:
- 완전한 isolation 아님 (호스트 커널 공유).
- 일부 K8s 기능 제한.

## 6. Cluster API — "K8s 클러스터를 K8s로 관리"

비유: Hard multi-tenancy를 하려면 클러스터를 자주 만듭니다. 그걸 또 GitOps로 선언적으로 관리. Cluster API가 이를 가능하게.

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata: { name: team-a-prod }
spec:
  clusterNetwork:
    pods: { cidrBlocks: [10.244.0.0/16] }
  controlPlaneRef:
    kind: KubeadmControlPlane
    name: team-a-control
  infrastructureRef:
    kind: AWSCluster
    name: team-a-aws
```

ApplicationSet(ArgoCD)으로 Cluster CR을 fan-out → 신규 팀에게 자동으로 클러스터 발급.

Tier 2 Course 10에서 자세히.

## 7. 빅테크 multi-tenancy 패턴

- **Tier 0 (regulated workload)**: 자체 클러스터 (hard).
- **Tier 1 (production service)**: 공유 클러스터 + namespace + 강력 NetworkPolicy + PSA restricted (soft).
- **Tier 2 (dev/CI)**: vCluster 또는 ephemeral namespace.

## 8. 운영 함정 5선

1. **Quota 미설정**: 한 팀이 노드 자원 독점.
2. **PSA 미설정**: 한 팀이 privileged Pod로 host 침해.
3. **NetworkPolicy 미설정**: 한 팀 Pod이 다른 팀 DB 접근.
4. **공유 클러스터에 root-equivalent CRD**: 한 팀이 cluster scope CRD로 다른 팀 영향.
5. **HNC 안 쓰고 namespace 수동 관리**: 정책 drift.

## 9. 실습

```bash
# 1. payments namespace + Quota + LimitRange
# 2. RBAC 으로 dev/lead 분리
# 3. default-deny NetworkPolicy
# 4. PSA restricted 라벨
# 5. vCluster 띄워 같은 호스트에 별도 K8s 체험
```

## 10. 자가평가 퀴즈

### Q1. Soft multi-tenancy의 본질적 한계?
1. **kernel 공유 — exploit 시 격리 깨짐**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q2. ResourceQuota 의 효과?
1. **namespace 자원 합계 한도 — 한 팀의 자원 독점 차단**
2. UI
3. 비용 절감
4. 무관

**정답: 1.**

### Q3. vCluster 의 핵심 가치?
1. **빠르고 안전한 cluster-admin 권한 분리(dev/test)**
2. 비용
3. UI
4. 무관

**정답: 1.**

### Q4. HNC 의 가치?
1. **namespace 부모-자식 정책 자동 상속**
2. 빠름
3. 비용
4. 무관

**정답: 1.**

### Q5. PCI/HIPAA 워크로드의 적절한 격리는?
1. **Hard — cluster 분리**
2. Soft
3. Virtual
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 12: 캡스톤]에서 12주 학습을 묶어 풀스택 multi-tenant 클러스터에 커스텀 operator를 배포합니다.
