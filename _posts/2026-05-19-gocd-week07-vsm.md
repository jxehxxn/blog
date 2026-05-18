---
layout: post
title: "GoCD Week 7: Value Stream Map + Pipeline Dependency"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd vsm
---

## 학습 목표

- VSM의 의미와 자동 생성.
- Pipeline dependency (upstream/downstream).
- Fan-in / Fan-out 패턴.
- 디버깅 도구로서 VSM.

## 1. 비유 — "공장 전체 흐름도 자동 그리기"

큰 자동차 공장: 엔진라인 → 차체라인 → 도장라인 → 최종 조립 → 검수. 각 단계가 서로 의존. 누군가 위에서 보면 한눈에 흐름 파악.

GoCD VSM이 이걸 **commit 1개 기준으로 자동 시각화**.

## 2. VSM 시각화

UI Pipeline → Value Stream Map 클릭.

```
[git commit abc123]
       ↓
[ build-pipeline ]  → artifact
       ↓
[ test-pipeline ] (pipeline material)
       ↓ 분기
[ deploy-dev ]  →  [ smoke-test ]
       ↓
[ deploy-stage ]
       ↓ manual approval
[ deploy-prod ]
```

각 node 클릭 → 해당 pipeline instance.

## 3. Pipeline Dependency

이미 4주차에서 다룸. material로 다른 pipeline 참조:

```yaml
materials:
  - pipeline:
      pipeline: build-pipeline
      stage: build
```

build-pipeline 성공 → 이 pipeline 자동 trigger. VSM에 화살표.

## 4. Fan-out (1 → N)

한 pipeline의 성공이 여러 downstream trigger:
```
[ build ]
   ↓
[deploy-us-east] [deploy-eu-west] [deploy-ap-ne]
```

같은 artifact를 3 region에 동시 배포. multi-region 표준.

## 5. Fan-in (N → 1)

여러 upstream의 모든 성공이 한 downstream trigger:
```
[ app-build ]  [ config-build ]
        ↓             ↓
        [ deploy-prod ]
```

deploy-prod에 두 pipeline material → 둘 다 성공해야 trigger.

GoCD가 가장 강력한 부분. **여러 material의 정확한 조합** trigger.

## 6. VSM 디버깅

"왜 prod에 어제 commit 안 갔지?" 같은 질문:
1. VSM 열기.
2. 시작 commit → 각 단계 결과 확인.
3. 어디서 fail/manual approval 대기 식별.
4. 클릭 → 상세.

5분 안에 답.

## 7. Material vs Pipeline Material 정확한 구분

- **Material (git)**: 외부 변경 → trigger.
- **Pipeline material**: 다른 pipeline 성공 → trigger.

둘 다 한 pipeline에 있을 수 있음.

## 8. 운영 함정 5선

1. **순환 dependency**: A → B → A. GoCD가 거부.
2. **너무 깊은 chain**: 10+ pipeline 의존. 디버깅 어려움.
3. **Fan-in 한쪽이 영원히 fail** → downstream 무한 대기.
4. **VSM 안 봄** → 사고 원인 추적 늦음.
5. **Pipeline rename**: dependency 깨짐.

## 9. 실습 과제

1. 3 pipeline: build → test → deploy.
2. VSM 확인 (linear).
3. deploy를 3 region으로 fan-out.
4. config repo material 추가 → fan-in 만들기.
5. 의도적 fail → VSM에서 어디 막힘 추적.

## 10. 자가평가 퀴즈

### Q1. VSM의 가치?
1. **commit이 production까지 가는 경로 자동 시각화**
2. UI 색상 3. 빠른 빌드 4. 무관

**정답: 1.**

### Q2. Fan-out?
1. **1 upstream → N downstream**
2. N → 1 3. UI 4. 무관

**정답: 1.**

### Q3. Fan-in?
1. **N upstream 모두 성공 → 1 downstream**
2. 1 → N 3. UI 4. 무관

**정답: 1.**

### Q4. 순환 dependency?
1. **GoCD가 거부**
2. 허용 3. UI 4. 무관

**정답: 1.**

### Q5. VSM 디버깅 가치?
1. **사고 원인 5분 추적**
2. UI 색 3. 무관 4. 빠른 빌드

**정답: 1.**

## 11. 다음 주차

[Week 8: Manual Approval + Trigger]에서 사람 결재와 trigger 종류.
