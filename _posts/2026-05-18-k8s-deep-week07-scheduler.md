---
layout: post
title: "K8s Deep Week 7: Scheduler 깊게 — Predicates, Priorities, 그리고 커스텀 스케줄러"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series scheduling
---

## 학습 목표

- Scheduler의 점수 알고리즘을 손으로 따라간다.
- Affinity / Anti-affinity / Topology Spread를 비유로 안다.
- Taints / Tolerations / nodeSelector의 차이를 안다.
- Scheduling Framework 플러그인을 작성한다.

## 1. 비유 — "수업 시간표 배정관"

대학 수업 배정관에 비유합니다.
1. 강의실 50개 중 (Filtering) "수강 인원이 들어갈 정도로 큰 곳"만 후보.
2. 남은 후보에 (Scoring) "프로젝터 있는 곳 +5, 교수님 연구실 근처 +3, 다른 수업과 충돌 안 함 +2" 같은 점수 부여.
3. 최고 점수 강의실 선택.

scheduler가 똑같이 합니다.

## 2. Filtering (예전 이름 Predicates)

대표 필터:
- `NodeUnschedulable`: cordon된 노드 제외.
- `PodFitsResources`: CPU/메모리 충분?
- `NodeAffinity`: nodeSelector / nodeAffinity 매칭?
- `PodAffinity`: 같이 두고 싶은 Pod 근처?
- `PodAntiAffinity`: 떨어뜨려야 할 Pod과 멀리?
- `TaintToleration`: taint를 견딜 toleration 있는가?
- `VolumeBinding`: 볼륨이 이 노드에서 attach 가능한가?
- `InterPodTopologySpread`: topology spread 제약?

하나라도 실패 → 후보 제외.

## 3. Scoring

남은 후보에 점수 (0~100 각 플러그인) 후 가중합.

주요:
- `NodeResourcesFit`: 자원 균형 / 최적 사용 정책.
- `NodeAffinity`: preferred affinity 매칭 점수.
- `ImageLocality`: 이미지가 이미 있는 노드 우대(이미지 pull 시간 절감).
- `InterPodAffinity`: preferred pod-affinity 점수.
- `NodeResourcesBalancedAllocation`: CPU/메모리 사용률 균형.
- `TaintToleration`: PreferNoSchedule taint 회피.

## 4. Affinity — "근처에 둬" vs "떨어뜨려"

비유: 항상 같이 다니는 친구(Pod) 들은 같은 강의실로 가도록, 으르렁대는 동아리(Pod) 들은 떨어진 강의실로.

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels: { app: webapp }
          topologyKey: kubernetes.io/hostname
```

같은 hostname(노드)에 다른 webapp Pod이 있으면 거기 두지 마. → 여러 노드에 분산.

`required`는 강제, `preferred`는 가산점. 운영에서는 보통 `preferred` 권장 (required는 schedule failure 위험).

## 5. Topology Spread Constraints — 좀 더 우아한 분산

비유: "캠퍼스 5개 동 건물에 학생들을 골고루 분산. 한 건물에 몰리지 마."

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels: { app: webapp }
```

`maxSkew: 1` — 가장 많은 zone과 가장 적은 zone의 차이가 1 이하여야 함. AZ 3개에 6 Pod = 각 2개씩.

빅테크 표준: AZ 분산 + 노드 분산 둘 다 적용.

## 6. Taints / Tolerations — "출입 통제"

비유: 강의실에 "교수 전용" 표지(taint)를 붙이면 일반 학생(Pod)이 못 들어옴. "교수 전용 토큰(toleration)"을 가진 사람만 출입 가능.

```bash
# 노드에 taint 추가
kubectl taint node gpu-1 dedicated=gpu:NoSchedule
```

```yaml
# Pod에 toleration
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
```

효과:
- `NoSchedule`: 새 Pod scheduling 거부 (기존 Pod 영향 없음).
- `PreferNoSchedule`: 가능하면 피함.
- `NoExecute`: 기존 Pod도 evict.

용도: GPU 노드를 GPU 워크로드만 쓰게, 노드 유지보수 시 cordon+drain, 특정 팀 전용 노드.

## 7. nodeSelector vs nodeAffinity

```yaml
# nodeSelector (단순)
spec:
  nodeSelector: { disktype: ssd }

# nodeAffinity (풍부)
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - { key: disktype, operator: In, values: [ssd, nvme] }
```

nodeAffinity가 더 표현력 풍부. nodeSelector는 legacy 호환.

## 8. Scheduling Framework — 커스텀 플러그인

K8s v1.19+ Scheduling Framework가 정식. 각 단계에 extension point.

```
PreFilter → Filter → PostFilter → PreScore → Score → NormalizeScore
→ Reserve → Permit → PreBind → Bind → PostBind
```

각 단계에 Go interface 구현해서 자신만의 로직 추가 가능.

### 예시: 비용 최적화 플러그인

```go
type CostAwarePlugin struct{}

func (c *CostAwarePlugin) Score(ctx context.Context, state *framework.CycleState,
  pod *v1.Pod, nodeName string) (int64, *framework.Status) {

    // 노드의 spot/ondemand 라벨 보고 점수 부여
    node := getNode(nodeName)
    if node.Labels["lifecycle"] == "spot" {
        return 80, framework.NewStatus(framework.Success, "")
    }
    return 20, framework.NewStatus(framework.Success, "")
}
```

빌드 → 별도 binary 실행 (`kube-scheduler --config=cost.yaml`) 또는 기본 scheduler에 통합.

빅테크 사례: Google이 PriorityClass 기반 preemption + 커스텀 score 플러그인으로 borg-like 스케줄링 구현.

## 9. PriorityClass + Preemption

비유: 교수 강의(높은 우선순위) 예약하면, 학생 동아리(낮은 우선순위) 예약을 밀어내고 들어감.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: { name: high }
value: 1000
preemptionPolicy: PreemptLowerPriority
```

priority가 높은 Pod이 unschedulable일 때 낮은 priority Pod을 evict해 자리 마련. 운영에서 매우 강력하지만 위험도 큼 — 함부로 high priority 남발하면 운영 불안정.

## 10. 실습

```bash
# 1. taint + toleration
kubectl taint node worker-3 special=true:NoSchedule
# 새 Pod 띄워보고 pending 확인
# Pod에 toleration 추가 → schedule

# 2. nodeAffinity preferred
# Pod에 preferred affinity 추가 후 logs 확인

# 3. topology spread
# replicas=6 deployment + spread by zone → kubectl get pod -o wide

# 4. 커스텀 plugin (Go) 작성 후 binary 실행
go build -o my-scheduler .
./my-scheduler --config=config.yaml
```

## 11. 자가평가 퀴즈

### Q1. Scheduler 2단계의 1단계는?
1. **Filtering — 조건 불만족 노드 제외**
2. Scoring
3. Eviction
4. Binding

**정답: 1.**

### Q2. `whenUnsatisfiable: DoNotSchedule` 의 위험?
1. **분산 제약을 못 맞추면 Pending 영구화**
2. 안전
3. 자동 무시
4. 무관

**정답: 1.**

### Q3. Taint NoExecute의 효과?
1. **기존 Pod도 evict, 새 Pod도 차단**
2. 새 Pod만 차단
3. 무관
4. 자동 무시

**정답: 1.**

### Q4. nodeSelector와 nodeAffinity 차이?
1. **nodeAffinity가 표현력 풍부, preferred 등 다층 정의**
2. 같은 것
3. nodeSelector가 더 빠름
4. 무관

**정답: 1.**

### Q5. Custom scheduler를 만드는 합리적 시점?
1. **기본 plugin 조합으로 못 표현하는 도메인 특화 정책 필요할 때**
2. 시작부터
3. 모든 경우
4. UI

**정답: 1.**

## 12. 다음 주차

[Week 8: Operator 개발]에서는 kubebuilder로 CRD + Controller를 처음부터 작성합니다.
