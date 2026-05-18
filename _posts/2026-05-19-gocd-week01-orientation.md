---
layout: post
title: "GoCD Week 1: 오리엔테이션 — GoCD와 Continuous Delivery 철학"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd
---

## 학습 목표

- GoCD의 출발과 ThoughtWorks의 영향.
- 다른 CI 도구와 다른 GoCD 철학 3가지.
- Pipeline-first vs Job-first 사고방식.
- 도입 가치 시점.

## 1. 비유로 시작 — "공장 컨베이어 vs 작업대"

Jenkins (전통 CI): 큰 작업대 + 도구들. 작업자가 job 하나씩 처리. 어디서 무엇이 막혔는지 위에서 보기 어려움.

GoCD: 컨베이어 벨트. 모든 부품(stage)이 어디서 시작해 어디로 가는지 **공장 전체 view**가 있고, 어디서 사람이 확인(manual gate)해야 하는지 표시됨.

이 비유가 GoCD를 한 줄로 설명합니다: **CD pipeline 전체를 1급 시민(first-class citizen)으로 모델링한 도구**.

## 2. 출발 — "Continuous Delivery" 책

GoCD의 모기업 ThoughtWorks의 Jez Humble & Dave Farley가 2010년 "Continuous Delivery" 책 출간 → 사실상 CD 운동의 출발. GoCD는 이 책의 철학을 도구로 구현한 것.

핵심 아이디어:
- 모든 변경이 **즉시 배포 가능한 상태**.
- Build pipeline이 변경 → 환경 흐름의 **명확한 그래프**.
- Manual approval은 의도된 게이트.
- Visibility (가시화) 최우선.

## 3. GoCD의 3 철학

### (a) Pipeline이 1급 시민
Jenkins의 Job, GitHub Actions의 Workflow와 비교해서, GoCD는 처음부터 "여러 단계가 흐르는 pipeline"을 중심으로 설계. UI도 pipeline 중심.

### (b) Value Stream Map (VSM)
한 commit이 어떤 pipeline들을 거쳐 production에 도달하는지 **자동 시각화**. 다른 도구에선 별도 구현해야 함.

### (c) Manual Approval은 기본
"이 stage는 사람 결재 필요" 가 도구 차원의 1등 개념. 빅테크 prod 배포에 자연스러움.

## 4. 다른 도구 한 줄 비교

| 도구 | 한 줄 |
|------|------|
| Jenkins | 가장 광범위, plugin 지옥, job 중심 |
| GitHub Actions | git 통합 강력, SaaS, workflow 중심 |
| Tekton | K8s native, Task/Pipeline 분리 |
| Argo Workflows | DAG 강력, ML/data 친화 |
| GoCD | **Pipeline + VSM + Approval gate**의 표준 |

## 5. 도입 가치 시점

- 변경 흐름이 복잡 (여러 pipeline → 1 prod).
- Audit/governance 강함 (manual approval 필요).
- 시각화 강력해야 (이해관계자 많음).
- 큰 기존 Java/legacy stack.

작은 팀 + 단순 CI는 GitHub Actions가 더 가벼움. GoCD는 CD 복잡도가 높을 때 가치 폭발.

## 6. 실제 사례

### ThoughtWorks 자체
사내 모든 client 프로젝트에 GoCD 표준.

### Financial / Healthcare
audit + manual approval 요구 → GoCD 자연. 일부 은행, 보험사 표준.

### Legacy Enterprise
Jenkins migration 대안. CD 가시화가 결정적.

## 7. 12주 로드맵

```
[기초] 1 오리엔테이션 → 2 설치 → 3 구조
[운영] 4 Materials → 5 Environment → 6 PaC
[심화] 7 VSM → 8 Approval → 9 Elastic Agent
[비교] 10 Plugins → 11 vs 다른 도구
[캡스톤] 12
```

## 8. 실습 과제 (필기형)

회사가 Jenkins로 50개 service CD 운영. "복잡해서 어떤 변경이 어디 가는지 모름" 문제. GoCD 도입을 30초 안에 임원에게 설득하려면 어떤 한 문장?

## 9. 자가평가 퀴즈

### Q1. GoCD의 모기업?
1. **ThoughtWorks**
2. Google 3. AWS 4. CNCF

**정답: 1.**

### Q2. GoCD가 영감을 받은 책?
1. **Continuous Delivery (Humble & Farley)**
2. Phoenix Project 3. DevOps Handbook 4. SRE Book

**정답: 1.**

### Q3. GoCD의 3 철학에 포함되지 않는 것?
1. Pipeline 1급 시민
2. Value Stream Map
3. Manual Approval 1급
4. **Plugin 우선**

**정답: 4.** plugin은 모든 CI에 있는 보편 기능.

### Q4. VSM (Value Stream Map)?
1. **commit이 production까지 가는 경로 자동 시각화**
2. UI 색상
3. 빠른 빌드
4. 무관

**정답: 1.**

### Q5. GoCD가 잘 맞는 환경?
1. **CD 복잡 + audit/approval 요구**
2. 1인 hobby project
3. 짧은 단일 빌드
4. 무관

**정답: 1.**

## 10. 다음 주차

[Week 2: 설치 + 첫 Pipeline]에서 GoCD server + agent를 띄우고 30분 안에 첫 pipeline을 봅니다.
