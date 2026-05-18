---
layout: post
title: "IaC Week 1: 오리엔테이션 — 콘솔 클릭의 종말과 IaC의 본질"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series
---

## 학습 목표

- 콘솔 운영의 한계(스노우플레이크, drift, audit 부재)를 안다.
- IaC의 4대 가치(declarative, idempotent, versioned, reviewable)를 안다.
- Terraform과 Crossplane이 같은 문제를 다른 방식으로 푼다는 것을 안다.

## 1. 비유 — "사주 카페" vs "건축 설계 사무소"

콘솔 클릭으로 인프라를 만드는 것: 사주 카페에 갈 때마다 "오늘은 좀 다른 결과네" 같은 느낌. 같은 클릭을 두 번 해도 미세한 차이로 결과가 갈리고, 한 달 후 "왜 이게 이렇게 됐지?" 추적이 어렵습니다.

IaC로 인프라를 만드는 것: 건축 설계 사무소. **도면(코드)** 이 있고, 도면대로 건물이 올라가며, 도면이 변경되면 그 변경이 기록(git)되고 검토(PR)됩니다. 도면을 다른 부지에 그대로 적용해 같은 건물을 다시 지을 수 있습니다(replicate).

## 2. 콘솔 운영의 7가지 죄

1. **스노우플레이크(Snowflake)**: 모든 서버가 미세하게 다름. 재현 불가.
2. **Drift**: 누군가 콘솔에서 만진 후 코드와 현실이 어긋남.
3. **Audit 부재**: "누가 언제 무엇을 바꿨는가" 추적 어려움.
4. **Disaster Recovery 부재**: 사고 시 같은 인프라 재구성 못 함.
5. **Onboarding 지옥**: 신입에게 "콘솔 100개 누르기" 가르쳐야 함.
6. **Multi-region 일관성 부재**: 5개 region을 모두 같이 만들기 거의 불가능.
7. **Cost 추적 불가**: 누가 무엇을 띄웠는지 모르고 비용만 상승.

이 7가지를 한 방에 푸는 것이 IaC.

## 3. IaC의 4대 가치

1. **Declarative**: 원하는 결과를 선언 (절차 X).
2. **Idempotent**: 100번 apply 해도 같은 결과.
3. **Versioned**: Git에 살고 history 추적.
4. **Reviewable**: PR + reviewer + diff.

## 4. Terraform — "범용 도면"

- HashiCorp의 IaC 도구. HCL이라는 자체 언어.
- 모든 클라우드(AWS, GCP, Azure, Alibaba, OCI...) provider 지원.
- 표준 워크플로: `init → plan → apply`.
- State 파일이 진실의 원천.

가장 많이 쓰이는 IaC, 사실상 업계 표준.

## 5. Crossplane — "K8s가 도면"

- CNCF 프로젝트. K8s 위에서 동작.
- **클라우드 리소스를 K8s CRD로 표현**.
- ArgoCD/Flux 같은 GitOps와 자연스럽게 결합.
- Composition으로 사내 표준 인프라 패키지화.

Crossplane은 "K8s를 universal control plane으로 쓰자"는 발상.

## 6. 둘은 경쟁인가, 보완인가

둘 다 IaC. 다른 철학.

| 항목 | Terraform | Crossplane |
|------|-----------|------------|
| State 위치 | 파일 (S3 등) | K8s etcd |
| 실행 방식 | CLI/CI 명령 (push) | K8s controller (pull, 지속 reconcile) |
| Drift 자동 감지 | 약함 (별도 plan 필요) | 강함 (continuous) |
| 학습 곡선 | 완만 | 가파름 (K8s 알아야) |
| 멀티 클라우드 추상 | provider 그대로 | XRD로 통합 추상 가능 |
| 빅테크 채택 | 매우 광범위 | 증가 추세 |

핵심: **현재의 표준은 Terraform, 미래의 표준 후보는 Crossplane.** 두 도구를 모두 알고, 시점·맥락에 맞게 고르는 능력이 시니어의 가치.

## 7. 실제 사례

### Netflix
- Terraform으로 AWS 인프라 거의 전체 운영. Atlantis로 PR 워크플로.
- 수만 개 모듈, 자체 Terraform CI 시스템.

### Adobe
- Crossplane으로 사내 "Adobe Tenant" 추상화. 1 CR이 namespace + IAM role + 사내 DB + 모니터링까지 자동 프로비저닝.
- 100+ 팀 셀프서비스.

### Upbound (Crossplane 모기업)
- Crossplane으로 멀티클라우드 universal control plane 비전.
- 사내에서 모든 인프라가 K8s CRD.

## 8. 12주 로드맵

```
[기초]    1주 오리엔테이션 → 2주 Terraform 기초 → 3주 State
              ↓
[운영]    4주 Modules → 5주 CI/Atlantis
              ↓
[전환]    6주 Crossplane 소개 → 7주 Composition → 8주 도구 비교
              ↓
[심화]    9주 Crossplane 운영 → 10주 Drift → 11주 거버넌스
              ↓
[캡스톤]  12주 멀티클라우드 풀스택
```

## 9. 실습 (필기형)

다음 시나리오: 회사가 콘솔로 AWS 운영하다가 5 region 확장을 결정. 한 팀이 "Terraform 도입" 다른 팀이 "Crossplane 도입" 주장. 어느 쪽을 어떤 근거로 추천하시겠습니까? 30초 설득 문장 작성.

## 10. 자가평가 퀴즈

### Q1. 콘솔 운영의 핵심 약점?
1. **스노우플레이크/drift/audit 부재 (재현불가)**
2. 비용
3. UI
4. 무관

**정답: 1.**

### Q2. IaC 4대 가치에 속하지 않는 것?
1. Declarative
2. Idempotent
3. Versioned
4. **Mutable (변경 가능)**

**정답: 4.**

### Q3. Terraform State 위치?
1. K8s etcd
2. **파일 (보통 S3/GCS 등 backend)**
3. Git
4. 무관

**정답: 2.**

### Q4. Crossplane의 차별점?
1. **클라우드 리소스를 K8s CRD로 표현 + continuous reconcile**
2. 빠른 CLI
3. UI
4. 무관

**정답: 1.**

### Q5. Adobe Tenant 같은 사내 추상화 패키지는 어느 도구가 더 자연스러운가?
1. Terraform
2. **Crossplane Composition**
3. 콘솔
4. 무관

**정답: 2.** Terraform 모듈로도 가능하지만 Composition + XRD가 더 자연스럽고 selfservice friendly.

## 11. 다음 주차

[Week 2: Terraform 기초]에서 `init/plan/apply`, provider, resource를 한 줄도 빠짐없이 봅니다.
