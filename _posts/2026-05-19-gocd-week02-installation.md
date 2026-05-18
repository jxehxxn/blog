---
layout: post
title: "GoCD Week 2: 설치 + 첫 Pipeline 30분 안에"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd
---

## 학습 목표

- GoCD server + agent 설치 (Docker / native / Helm).
- 첫 pipeline 생성.
- UI 핵심 화면 5개.

## 1. 비유 — "서버 (지휘자) + 에이전트 (연주자)"

GoCD 아키텍처:
- **Server**: pipeline 정의·스케줄·UI·artifact 보관.
- **Agent**: 실제 build/test 실행 worker. 노트북·VM·K8s Pod 등.

지휘자(server)가 악보(pipeline) 배포, 연주자(agent)가 연주(build).

## 2. Docker 설치 (가장 빠름)

```bash
# Server
docker run -d --name gocd-server \
  -p 8153:8153 -p 8154:8154 \
  -v gocd-server-data:/godata \
  gocd/gocd-server:v25.x

# Agent
docker run -d --name gocd-agent \
  -e GO_SERVER_URL=https://gocd-server:8154/go \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gocd/gocd-agent-docker-dind:v25.x
```

UI: `http://localhost:8153/go`.

첫 화면에서 admin 계정 만듭니다.

## 3. Helm 설치 (K8s)

```bash
helm repo add gocd https://gocd.github.io/helm-chart
helm install gocd gocd/gocd \
  -n gocd --create-namespace \
  --set server.service.type=LoadBalancer
```

K8s 환경의 빅테크 표준.

## 4. 첫 Pipeline (UI)

1. Login.
2. "Admin → Pipelines → Create New Pipeline".
3. Material: Git URL 입력 (예: `https://github.com/gocd/gocd-demo-app`).
4. Stage: `build`.
5. Job: `compile`.
6. Task: shell `mvn clean install`.
7. Save.

자동으로 첫 build 시작. UI에서 진행 상황 확인.

## 5. 핵심 UI 화면 5개

### (a) Pipelines (홈)
모든 pipeline의 최근 status 한눈에. 녹/적/회색 (success/fail/idle).

### (b) Pipeline Details
한 pipeline의 stage 순서, 시간, 결과.

### (c) Value Stream Map (VSM) ⭐
한 commit이 어떤 pipeline들을 거쳐 production까지 가는 시각화. GoCD의 시그니처.

### (d) Console Output
각 job의 실시간 build 로그.

### (e) Admin
pipeline / agent / environment / user 설정.

## 6. Agent 등록

새 agent가 server에 처음 연결 시 "Pending" 상태. Admin에서 "Approve" 클릭 → 활성. 보안.

빅테크 표준: auto-register key로 자동 승인.

## 7. CLI

```bash
# CLI 도구 (선택)
brew install gocd-cli
gocd config --server http://localhost:8153 --user admin --password admin
gocd pipeline list
gocd pipeline trigger my-pipeline
```

## 8. 실습 과제

1. Docker로 server + agent 설치.
2. github의 sample (`gocd/gocd-demo-app`) 로 첫 pipeline.
3. build 결과를 UI에서 확인.
4. 의도적으로 build fail 만들기 (잘못된 dep 등) → UI 표시 확인.
5. agent를 임시 멈춰 pending 상태 보기.

## 9. 자가평가 퀴즈

### Q1. GoCD의 2 컴포넌트?
1. **Server + Agent**
2. Master + Worker (Jenkins 용어)
3. Controller + Runner
4. Hub + Spoke

**정답: 1.** (다른 용어와 비슷하지만 정확히 Server/Agent.)

### Q2. UI 시그니처 화면?
1. **Value Stream Map**
2. Logs
3. Settings
4. Dashboard

**정답: 1.**

### Q3. 새 Agent의 첫 상태?
1. **Pending (수동 approve 필요)**
2. Active
3. Idle
4. Error

**정답: 1.**

### Q4. K8s 환경 표준 설치?
1. **Helm chart**
2. manual yaml
3. apt-get
4. exe

**정답: 1.**

### Q5. Auto-register key 효과?
1. **신규 agent 자동 approve**
2. UI 색상
3. 빠른 빌드
4. 무관

**정답: 1.**

## 10. 다음 주차

[Week 3: Pipeline / Stage / Job / Task 구조]에서 GoCD의 4 계층 구조를 한 줄도 빠짐없이 풉니다.
