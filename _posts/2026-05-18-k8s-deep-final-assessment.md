---
layout: post
title: "K8s Deep 최종 자가평가 — 30문항 종합 퀴즈 (정답+해설)"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series assessment
---

12주를 마친 학습자의 종합 평가. 4지선다 30문항, 정답 + 해설.

## 섹션 A — 컨트롤·데이터 플레인 (Q1~Q10)

### Q1. Kubernetes 한 줄 정의?
1. 컨테이너 빌드 도구
2. **클러스터를 한 OS로 다루는 분산 OS**
3. CI/CD 시스템
4. 모니터링 도구

**정답: 2.**

### Q2. etcd 가 홀수 노드여야 하는 이유?
1. 짝수면 동작 안 함
2. **quorum 계산이 명확 (3 중 2, 5 중 3)**
3. 비용
4. 무관

**정답: 2.**

### Q3. API server 의 4단계 처리 순서?
1. **Auth → Authz → Admission → Storage**
2. Storage → Auth → Authz → Admission
3. 무순서
4. 의미 없음

**정답: 1.**

### Q4. Scheduler 의 2단계?
1. **Filtering → Scoring**
2. Scoring → Filtering
3. Binding → Eviction
4. 무관

**정답: 1.**

### Q5. Controller-manager 핵심 패턴?
1. **Reconciliation (desired vs actual 비교)**
2. Polling
3. Push
4. cron

**정답: 1.**

### Q6. Controller 가 변화를 감지하는 방식?
1. polling
2. **Watch (long HTTP)**
3. webhook
4. cron

**정답: 2.**

### Q7. CRI 의 약자?
1. **Container Runtime Interface**
2. Container Reduced Image
3. 약자 아님
4. 무관

**정답: 1.**

### Q8. "컨테이너" 의 본질?
1. 작은 VM
2. **Linux namespace + cgroup 으로 격리·제한된 일반 프로세스**
3. Docker 전용
4. JVM

**정답: 2.**

### Q9. Pod 안 두 컨테이너가 localhost 통신 가능한 이유?
1. **같은 network namespace 공유 (pause 컨테이너)**
2. NAT
3. iptables
4. 무관

**정답: 1.**

### Q10. kube-proxy IPVS 모드의 장점?
1. UI
2. **수천 Service 환경에서 iptables 대비 성능 우수**
3. 비용
4. 무관

**정답: 2.**

## 섹션 B — 네트워크 / 스토리지 / 스케줄링 (Q11~Q20)

### Q11. K8s 네트워크 3원칙 중 가장 중요한 것?
1. **모든 Pod이 unique IP + NAT 없이 직접 통신**
2. 모든 Pod 같은 IP
3. NAT 필수
4. 무관

**정답: 1.**

### Q12. NetworkPolicy enforcement 주체?
1. API server
2. **CNI (Calico/Cilium)**
3. kubelet
4. scheduler

**정답: 2.**

### Q13. AWS EBS가 RWX 안 되는 이유?
1. **블록 디바이스는 본질적으로 한 호스트만 mount 가능**
2. AWS 정책
3. K8s 한계
4. 무관

**정답: 1.**

### Q14. StatefulSet volumeClaimTemplates 효과?
1. **각 Pod에 자기 PVC 자동 (재시작해도 같은 볼륨)**
2. 같은 볼륨
3. UI
4. 무관

**정답: 1.**

### Q15. WaitForFirstConsumer 의 가치?
1. **Pod 스케줄 후 볼륨 생성 → AZ 일치**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q16. PSA restricted 의 효과?
1. **runAsNonRoot, drop ALL capabilities 강제**
2. 무제한
3. UI
4. 무관

**정답: 1.**

### Q17. Topology Spread maxSkew 의 의미?
1. **가장 많은 / 가장 적은 zone 차이 한도**
2. 무관
3. 클러스터 크기
4. 비용

**정답: 1.**

### Q18. Taint NoExecute 효과?
1. **기존 Pod도 evict, 새 Pod 차단**
2. 새 Pod 만 차단
3. 무관
4. 자동 무시

**정답: 1.**

### Q19. Cluster Autoscaler 의 트리거?
1. **Pending Pod 발생 시 노드 증설**
2. CPU 사용량
3. 메모리
4. 무관

**정답: 1.** (Karpenter는 좀 더 다양한 신호 — Tier 2)

### Q20. Custom scheduler가 합리적인 시점?
1. **기본 plugin 으로 못 표현하는 도메인 정책 필요**
2. 시작부터
3. 모든 경우
4. UI

**정답: 1.**

## 섹션 C — Operator / 보안 / 성능 (Q21~Q30)

### Q21. Operator의 본질?
1. **도메인 지식을 controller로 코드화 + CRD**
2. 단순 자동화
3. UI
4. 무관

**정답: 1.**

### Q22. Owner Reference 효과?
1. **부모 삭제 시 자식 cascading delete**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q23. Finalizer 용도?
1. **삭제 직전 외부 자원 정리**
2. 생성 직전
3. UI
4. 무관

**정답: 1.**

### Q24. RBAC subject 3종?
1. **User, Group, ServiceAccount**
2. Pod/Service/Deployment
3. namespace
4. 무관

**정답: 1.**

### Q25. Secret 의 etcd 저장 (기본)?
1. AES-256
2. **base64 인코딩만 (사실상 평문)**
3. PGP
4. 무관

**정답: 2.**

### Q26. APF 의 핵심 가치?
1. **요청 카테고리화로 한 사용자 폭주가 전체에 영향 안 가게**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q27. etcd defrag 안 하면?
1. **DB 파일 부풀어 성능 저하, quota 초과**
2. 안전
3. 무관
4. 비용

**정답: 1.**

### Q28. CoreDNS 폭주 흔한 원인?
1. **Pod ndots: 5 기본값으로 lookup 폭증**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q29. ResourceQuota 효과?
1. **namespace 자원 합계 한도**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q30. PCI/HIPAA 워크로드 적절 격리?
1. **Hard — 클러스터 분리**
2. Soft
3. Virtual
4. 무관

**정답: 1.**

## 채점

- 28~30: 빅테크 Senior Platform 수준.
- 24~27: 미드 레벨, 성능·보안 추가 학습.
- 18~23: 주니어, operator·multi-tenancy 보완.
- ~17: 핵심 챕터(2,3,4,7,8) 재복습.

수고하셨습니다. 다음 코스에서는 이 위에 Terraform + Crossplane을 얹어 **인프라 자체를 GitOps**로 만듭니다.
