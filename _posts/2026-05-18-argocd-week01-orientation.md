---
layout: post
title: "ArgoCD Week 1: 오리엔테이션 — 왜 GitOps인가, 왜 ArgoCD인가"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes
---

## 학습 목표

- GitOps의 4대 원칙을 정확한 용어로 이해한다.
- Push 배포(Jenkins, Spinnaker)와 Pull 배포(ArgoCD, Flux)의 본질적 차이를 안다.
- ArgoCD가 Flux 같은 경쟁 도구 대비 가지는 포지셔닝을 비교한다.
- 12주 학습 로드맵을 시각화한다.

## 1. 비유로 시작 — "원격 배달원" vs "스스로 챙겨가는 직원"

비유합니다. Push 모델은 **배달원이 회사 정문 안까지 들어와 책상에 코드를 놓고 가는 것**입니다. 누가 들어왔는지, 무엇을 두고 갔는지 정문 보안만 알 수 있고, 내부에서 일이 잘못되면 책임 추적이 어렵습니다.

Pull 모델은 **각 부서가 정문 우편함을 주기적으로 확인하고, 자기 앞으로 온 우편물을 직접 가져가는 것**입니다. 부서 안에서 무엇이 도착했고 어떻게 처리했는지 명확하게 추적되며, 외부에서 부서 내부를 직접 만질 수 없습니다.

Kubernetes 운영의 보안 모델과 정확히 일치합니다.
- Push: CI가 클러스터 자격증명을 가지고 `kubectl apply`. **클러스터 자격증명이 CI 시스템에 산다.**
- Pull: 클러스터 안의 ArgoCD가 Git을 폴링해 자체적으로 동기화. **외부로 자격증명이 나가지 않는다.**

## 2. GitOps의 4대 원칙 (CNCF 정의)

1. **Declarative**: 원하는 상태(desired state)를 선언형으로 표현한다.
2. **Versioned and Immutable**: 그 선언이 Git 같은 버전관리에 변경 불가능하게 저장된다.
3. **Pulled Automatically**: 시스템이 그 선언을 자동으로 가져와 적용한다.
4. **Continuously Reconciled**: 실제 상태가 선언 상태로 끊임없이 수렴한다.

이 4가지가 빠진 어떤 것도 GitOps가 아닙니다. "Jenkins로 kubectl apply 자동화"는 1, 2번은 있을 수 있지만 3, 4번이 없어서 GitOps가 아닙니다.

## 3. Push vs Pull — 정량 비교

| 항목 | Push (Jenkins/Spinnaker) | Pull (ArgoCD/Flux) |
|------|--------------------------|---------------------|
| 클러스터 자격증명 위치 | CI 시스템 | 클러스터 안 |
| 외부→내부 네트워크 | 필요 (CI가 API 접근) | 불필요 (내부→외부 git pull만) |
| Drift 자동 감지 | 없음 (배포 후 모름) | 있음 (continuous reconcile) |
| Rollback | 별도 파이프라인 | git revert + auto sync |
| Audit | CI 로그에 흩어짐 | Git history + ArgoCD audit |
| Multi-cluster fan-out | 어려움 | ApplicationSet으로 자동 |

빅테크가 Pull로 갈아탄 가장 큰 이유는 **공격 표면 축소**입니다. CI 시스템 침해가 곧 production 클러스터 침해로 직결되는 것을 끊을 수 있습니다.

## 4. 실제 사례 3선

### 사례 1: Intuit의 ArgoCD 채택과 기여

Intuit은 ArgoCD의 모체이자 가장 큰 contributor입니다. TurboTax 등 수만 개 마이크로서비스를 5,000+ 클러스터에 배포하는데, Jenkins push 모델로는 1) 보안 자격증명 분산, 2) drift 추적 불가, 3) 멀티클러스터 fan-out 부재의 3중 문제가 있었습니다. ArgoCD로 전환 후 평균 배포 시간 1/3, 배포 실패 시 평균 복구 시간 1/5로 단축된 것이 KubeCon 사례 발표에 등장했습니다.

### 사례 2: Adobe Experience Platform

Adobe는 Spinnaker push 기반 → ArgoCD pull 기반으로 전환하면서 클러스터당 1명이던 운영 부담을 1명이 50 클러스터를 다루는 수준으로 줄였다고 발표했습니다. 핵심은 **선언적 + 멀티클러스터 fan-out + 자동 reconcile** 세 가지의 결합.

### 사례 3: Spotify의 Backstage + ArgoCD

Spotify는 자체 IDP(Internal Developer Platform) Backstage와 ArgoCD를 연결해 개발자가 PR만 머지하면 production까지 가는 워크플로를 만들었습니다. 개발자는 클러스터를 모르고, 플랫폼팀은 ArgoCD만 관리.

## 5. ArgoCD vs Flux — 같은 GitOps, 다른 철학

| 항목 | ArgoCD | Flux v2 |
|------|--------|---------|
| UI | 강력한 Web UI | CLI 중심, Web UI 별도 |
| Application 정의 | Application CRD 단일 | Source/Kustomization/HelmRelease 분리 |
| Multi-tenant | Projects + AppProject | Tenancy via namespace+RBAC |
| ApplicationSet | 강력한 generator 8종 | 비슷 (Flagger 별도) |
| Progressive delivery | Argo Rollouts | Flagger |
| 학습 곡선 | 비교적 완만 (UI 도움) | 약간 가파름 (CLI/CRD) |

빅테크는 두 도구 모두 사용합니다만, **UI 기반 협업이 필요한 큰 조직**은 ArgoCD를, **GitOps 순수성·CRD 모듈성**을 선호하면 Flux를 고릅니다.

## 6. 12주 로드맵

```
[기초]   1주 오리엔테이션 → 2주 첫 sync → 3주 Application/Set
              ↓
[운영]   4주 Sync 전략 → 5주 리포지토리 도구
              ↓
[격리]   6주 멀티클러스터 → 7주 RBAC/Project
              ↓
[심화]   8주 시크릿 → 9주 Progressive Delivery → 10주 Observability
              ↓
[관리]   11주 거버넌스/DR
              ↓
[캡스톤] 12주 풀스택 시뮬레이션
```

## 7. 실습 과제 (필기형)

다음 시나리오를 읽고 답하세요.

**시나리오:** 회사가 매주 1회 새벽 배포 윈도우(deployment window)를 두고 Jenkins로 kubectl apply 하고 있습니다. 매니저가 "GitOps로 전환을 검토하라"고 했습니다. 임원에게 30초 안에 "왜 전환해야 하는가"를 설명하려면 어떤 한 문장을 던지겠습니까? (Intuit, Adobe, Spotify 사례 중 최소 하나 인용)

## 8. 자가평가 퀴즈

### Q1. GitOps 4대 원칙에 포함되지 않는 것은?
1. Declarative
2. Versioned and Immutable
3. Continuously Reconciled
4. **Push로 적용**

**정답: 4.** 자동으로 Pull. Push는 GitOps의 반대 개념.

### Q2. Push 모델의 가장 큰 보안 약점은?
1. CI가 클러스터 자격증명을 가져 CI 침해가 클러스터 침해로 직결
2. 빌드가 느리다
3. UI가 없다
4. YAML이 어렵다

**정답: 1.**

### Q3. ArgoCD가 Flux보다 빅테크에서 더 자주 선택되는 이유는?
1. UI가 강력해 협업·디버깅이 쉬움
2. CLI가 더 좋아서
3. 더 빠르다
4. 무료이기 때문

**정답: 1.** (둘 다 OSS이고 둘 다 빠릅니다.)

### Q4. Drift 감지가 GitOps의 핵심 가치 중 하나인 이유는?
1. 누군가 클러스터를 직접 만져도 Git에 정의된 desired state로 자동 수렴
2. 디스크 용량 절약
3. 빌드 속도 향상
4. 의미 없음

**정답: 1.**

### Q5. Spotify가 Backstage + ArgoCD를 결합한 핵심 가치는?
1. 개발자는 클러스터를 몰라도 PR 머지로 production까지 도달
2. ArgoCD UI가 예뻐서
3. Backstage 라이선스 비용 절감
4. 의미 없음

**정답: 1.**

## 9. 다음 주차

[Week 2: ArgoCD 기초]에서는 직접 클러스터에 ArgoCD를 설치하고, 30분 안에 첫 Application을 sync합니다.
