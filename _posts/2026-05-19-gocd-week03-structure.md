---
layout: post
title: "GoCD Week 3: Pipeline / Stage / Job / Task — 4계층 구조"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd
---

## 학습 목표

- 4계층 구조와 각 계층의 책임.
- 순차(stage)와 병렬(job)의 차이.
- Task 종류 (exec, ant, rake, fetch artifact, plugin).
- 흔한 함정.

## 1. 비유 — "조립 라인의 4단계 위계"

자동차 공장:
- **Pipeline** = 완성차 1대를 만드는 전체 라인.
- **Stage** = 큰 단계 (도장 → 조립 → 검수). 순차.
- **Job** = 한 단계 안의 작업단위 (도장 라인 안의 천장/문/문틀). 병렬.
- **Task** = 한 job 안의 구체 명령 (붓 잡기 → 칠하기 → 닦기). 순차.

## 2. 구조 그림

```
Pipeline
├─ Stage 1: build
│  ├─ Job: compile
│  │   ├─ Task: mvn clean
│  │   └─ Task: mvn compile
│  └─ Job: lint  (병렬)
│      └─ Task: checkstyle
├─ Stage 2: test
│  ├─ Job: unit-test
│  └─ Job: integration-test (병렬)
└─ Stage 3: deploy (manual approval)
   └─ Job: kubectl-apply
       └─ Task: kubectl apply -f ...
```

핵심:
- Stage는 **순차**. test stage는 build stage 완료 후.
- 같은 stage 안 Job은 **병렬**. unit-test와 integration-test 동시.
- Job 안 Task는 **순차**. mvn clean → mvn compile.

## 3. Task 종류

### Exec
가장 흔함. shell 명령:
```yaml
- exec:
    command: bash
    args: ["-c", "make build"]
```

### Ant / Rake / Nant
빌드 도구 native:
```yaml
- ant:
    buildFile: build.xml
    target: compile
```

### Fetch Artifact
이전 stage/pipeline의 결과물 가져오기:
```yaml
- fetch:
    pipeline: upstream
    stage: package
    job: build
    source: dist/app.jar
    destination: ./
```

이게 GoCD의 강력함: **artifact를 명시적으로 stage 간 전달**.

### Plugin Task
외부 plugin (예: Docker, Kubernetes deploy).

## 4. Stage 정책

### Trigger
- **On Success**: 이전 stage 성공하면 자동.
- **Manual**: 사람이 클릭해야 시작.
- **Success with manual approval**: 성공해도 사람 결재.

```yaml
stages:
  - name: build
    approval: success    # 기본
  - name: deploy-prod
    approval:
      type: manual
      authorization:
        roles: [prod-admin]
```

빅테크 prod stage 표준: manual + RBAC.

## 5. Job 옵션

```yaml
jobs:
  - name: compile
    elastic_profile_id: docker-jdk17    # Elastic Agent
    timeout: 30                          # 분
    artifacts:
      - build:
          source: target/*.jar
          destination: dist/
    resources: [linux, jdk17]            # Agent 매칭 라벨
```

resources로 적합한 agent 자동 선택.

## 6. Fan-in (multiple materials)

여러 material 변경이 한 pipeline trigger:
```yaml
materials:
  - git:
      url: https://github.com/mycorp/app.git
  - git:
      url: https://github.com/mycorp/config.git
```

둘 다 polling. 한 쪽 변경에 trigger. 4주차에서 깊게.

## 7. Artifact

stage 끝나면 자동 archive (server에 보관). 다음 stage/pipeline이 fetch.

종류:
- **Build artifact**: 빌드 결과 (jar, image tag).
- **Test artifact**: test report.

UI에서 download 가능.

## 8. 실전 함정 5선

1. **Stage 너무 많이**: 5개 이상은 추적 어려움.
2. **Job 안에 너무 많은 task**: 실패 시 어디인지 모름. 분리.
3. **Artifact 너무 크게**: server disk 폭증. retention 필수.
4. **Manual approval 누락**: prod 자동 배포 사고.
5. **resources 매칭 잘못**: agent 부족 → 작업 hang.

## 9. 실습 과제

1. 3 stage pipeline 만들기: build → test → deploy (manual).
2. test stage에 2 job 병렬 (unit + integration).
3. build의 artifact를 deploy에서 fetch.
4. deploy에 manual approval + role.
5. 의도적으로 build 실패 → 다음 stage 차단 확인.

## 10. 자가평가 퀴즈

### Q1. 4계층 순서?
1. **Pipeline > Stage > Job > Task**
2. Workflow > Job > Step > Action
3. Pipeline > Job > Stage > Task
4. 무관

**정답: 1.**

### Q2. Stage 간 관계?
1. **순차 (이전 성공 시 다음)**
2. 병렬
3. 무관
4. 임의

**정답: 1.**

### Q3. Job 간 관계 (같은 stage 안)?
1. **병렬**
2. 순차
3. 무관
4. 무작위

**정답: 1.**

### Q4. Fetch Artifact의 가치?
1. **stage/pipeline 간 산출물 명시적 전달**
2. UI 색상
3. 빠름
4. 무관

**정답: 1.**

### Q5. Manual approval 표준 적용?
1. **prod stage + RBAC role**
2. 모든 stage
3. UI
4. 무관

**정답: 1.**

## 11. 다음 주차

[Week 4: Materials]에서 Git/SCM 통합과 multi-material, polling 동작을 다룹니다.
