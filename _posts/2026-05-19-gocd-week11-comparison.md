---
layout: post
title: "GoCD Week 11: vs Jenkins / GitHub Actions / Tekton — 의사결정"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd comparison
---

## 학습 목표

- 6 영역 정량 비교.
- 시나리오별 권장.
- 마이그레이션 시점.

## 1. 6 영역 비교

| 항목 | GoCD | Jenkins | GH Actions | Tekton |
|------|------|---------|------------|--------|
| 호스팅 | 자체 | 자체 | SaaS | 자체 (K8s) |
| Pipeline 모델 | First-class | Job 중심 | Workflow | Pipeline (K8s native) |
| VSM | **내장** | plugin | 없음 | 없음 |
| Manual Approval | **first-class** | plugin | environments | 없음 |
| Multi-material fan-in | **내장** | plugin | 약함 | 약함 |
| K8s 통합 | plugin | plugin | self-hosted runner | **native** |
| 학습 곡선 | 중간 | 가파름 (plugin) | 완만 | 가파름 (K8s) |
| 빅테크 사용 | finance/legacy | 광범위 (legacy) | OSS/startup | 신규 |

## 2. 시나리오별 권장

### CD 흐름 복잡 + audit/approval 중요
**GoCD**. VSM + manual approval first-class.

### 기존 Jenkins, 점진 개선
**Jenkins** 유지 + GoCD 평가.

### Public OSS / GitHub 중심
**GitHub Actions**.

### K8s native 환경
**Tekton** (Course 3).

### 가벼운 startup
**GH Actions**.

## 3. GoCD의 강점

1. VSM (자동 시각화).
2. Manual approval RBAC.
3. Multi-material fan-in 자연.
4. Pipeline의 1급 시민화.

## 4. GoCD의 약점

1. Cloud-native 도구 대비 작은 community.
2. K8s 통합은 plugin 의존.
3. UI가 modern 도구 대비 떨어짐.
4. Marketplace 작음.

## 5. 마이그레이션 시점

- 기존 Jenkins job 100+ 이상 + CD 가시화 필요 → GoCD 평가.
- 단순 CI → GH Actions / GitLab CI.
- 컨테이너 native → Tekton.

## 6. 빅테크 사례

- **금융권 일부**: GoCD 표준 (audit + approval).
- **ThoughtWorks 자체**: 모든 client.
- **Enterprise legacy**: Jenkins 잔존, GoCD 일부 도입.

## 7. 자가평가 퀴즈

### Q1. GoCD 강점 1?
1. **VSM 내장**
2. 가벼움 3. SaaS 4. UI

**정답: 1.**

### Q2. CD 복잡 + audit?
1. **GoCD**
2. GH Actions 3. Tekton 4. 무관

**정답: 1.**

### Q3. K8s native?
1. **Tekton**
2. GoCD 3. Jenkins 4. UI

**정답: 1.**

### Q4. 가벼운 startup?
1. **GH Actions**
2. GoCD 3. Jenkins 4. Tekton

**정답: 1.**

### Q5. 빅테크 GoCD 채택 영역?
1. **금융권 + ThoughtWorks**
2. 게임 3. 무관 4. 모두

**정답: 1.**

## 8. 다음 주차

[Week 12: Capstone].
