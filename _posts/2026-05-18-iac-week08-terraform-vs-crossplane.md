---
layout: post
title: "IaC Week 8: Crossplane vs Terraform — 실제 의사결정 기준"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series
---

## 학습 목표

- 두 도구의 본질적 차이를 한 줄로 정리한다.
- 8개 영역에서 정량 비교.
- 시나리오별 권장 도구를 결정한다.
- 하이브리드 운영 패턴을 이해한다.

## 1. 한 줄 정리

- **Terraform**: 외부에서 클라우드를 push로 조작. 범용·생태계.
- **Crossplane**: K8s 안에서 클라우드를 pull로 reconcile. GitOps·추상화.

## 2. 8 영역 비교

| 항목 | Terraform | Crossplane |
|------|-----------|------------|
| 실행 모델 | CLI/CI push | K8s controller pull |
| State 위치 | 파일 (S3 등) | K8s etcd |
| Drift 자동 감지 | 약함 (별도 plan) | 강함 (continuous) |
| Continuous reconcile | 없음 | 있음 |
| Provider 수 | 3,000+ | 100+ (성장 중) |
| 학습 곡선 | 완만 | 가파름 (K8s 필요) |
| 사내 추상화 | 모듈 | XRD + Composition (더 강력) |
| Multi-IaC 결합 | 어려움 | K8s 위 ArgoCD로 자연스럽게 |

## 3. 시나리오별 권장

### 시나리오 1: 스타트업 첫 인프라
- **Terraform 추천**. 학습 비용 낮고, K8s 클러스터가 아직 없을 수 있음.

### 시나리오 2: 이미 K8s 운영, GitOps 도입 중
- **Crossplane 추천**. ArgoCD와 자연스러운 결합, 추상화 강력.

### 시나리오 3: 멀티 클라우드 (AWS + GCP + Azure)
- **둘 다 가능**. Crossplane이 XRD로 cloud 추상화에 약간 유리.

### 시나리오 4: SaaS 앱팀에 셀프서비스 인프라
- **Crossplane 추천**. Claim 모델 = K8s native selfservice.

### 시나리오 5: 기존 거대 Terraform monorepo
- **Terraform 유지** + 새 인프라만 Crossplane으로 점진 이주.

### 시나리오 6: 인프라 자원이 거의 Static (10년 변경 없음)
- **Terraform** + 1회 apply. continuous reconcile 가치 작음.

### 시나리오 7: Drift가 자주 문제됨 (콘솔 직접 변경 잦음)
- **Crossplane 추천**. 자동 복구가 핵심 가치.

## 4. 하이브리드 패턴

빅테크 다수가 둘을 함께 운영:

```
[ Bottom: Terraform ]
  - VPC, IAM, EKS cluster 자체 같은 기반 인프라
  - 거의 안 바뀌는 자원
  - K8s 자체가 없으므로 K8s 의존성 못 만듦

[ Top: Crossplane (K8s 위) ]
  - 앱별 DB, S3, IAM role
  - 자주 변경 + selfservice 필요
  - ArgoCD GitOps로 통합 관리
```

자세한 패턴은 [보충 4: Terraform-Crossplane 하이브리드].

## 5. State 위치 트레이드오프

| 항목 | Terraform (S3) | Crossplane (etcd) |
|------|----------------|--------------------|
| 백업 | S3 versioning + cross-region | etcd snapshot |
| 손실 시 위험 | 동일 (자원 재생성/destroy 위험) | 동일 |
| Access 제어 | S3 IAM | K8s RBAC |
| Audit | S3 access log | K8s audit log |

둘 다 백업·access·audit 정책 필요. 도구 선택보다 운영 규율이 본질.

## 6. 성능

- Terraform: 큰 monorepo는 plan 5~30분 가능. 분할 필요.
- Crossplane: 자원 수 증가 시 controller reconcile 부하. sharding/optimization 필요.

둘 다 1만+ 자원에서는 운영 노하우 필요.

## 7. 사람·조직 요인

빅테크 환경:
- DevOps 엔지니어 다수가 Terraform 경험. Crossplane은 K8s 전문가 필요.
- Crossplane으로 가려면 platform 팀의 K8s 운영 역량이 필수.
- 둘 다 익혀 두면 시니어로서 가치 큼.

## 8. 미래

CNCF 트렌드: Crossplane 채택 증가, OpenTofu(Terraform fork) 활발. 둘 다 활용 시점.

빅테크 다수: "Terraform 유지 + Crossplane 점진 도입" 이 현실적 답.

## 9. 실습 (필기형)

다음 시나리오를 읽고 어느 도구 또는 조합을 추천할지 200자로:

1. 5명 스타트업, AWS 처음 도입.
2. 1,000명 빅테크, 멀티클라우드, K8s 운영 중, 앱팀 셀프서비스 필요.
3. 금융 회사, 거의 안 바뀌는 핵심 인프라, 변경 시마다 감사.

## 10. 자가평가 퀴즈

### Q1. Terraform vs Crossplane 본질 차이?
1. UI
2. **실행 모델 (push vs pull) + State 위치**
3. 가격
4. 무관

**정답: 2.**

### Q2. 콘솔 직접 변경(drift)이 잦은 환경에 적합?
1. **Crossplane (continuous reconcile)**
2. Terraform
3. 무관
4. 콘솔만

**정답: 1.**

### Q3. 스타트업 첫 인프라에 적합?
1. **Terraform (학습 비용 낮음, K8s 의존성 없음)**
2. Crossplane
3. 콘솔
4. 무관

**정답: 1.**

### Q4. 빅테크 하이브리드 패턴의 일반 구도?
1. **Bottom Terraform (기반), Top Crossplane (앱별 자원)**
2. 반대
3. Terraform 만
4. Crossplane 만

**정답: 1.**

### Q5. Crossplane이 ArgoCD와 잘 맞는 이유?
1. **인프라가 K8s 객체라 ArgoCD sync 그대로 적용**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 9: Crossplane 운영]에서 providers, packages, RBAC 운영을 다룹니다.
