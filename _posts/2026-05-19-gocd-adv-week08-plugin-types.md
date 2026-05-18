---
layout: post
title: "GoCD 심화 Week 8: Plugin Type별 개발"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd plugin senior
---

## 1. Task Plugin

```java
@Extension
public class MyTaskPlugin implements GoPlugin {
    // request: "go.plugin-settings.get-view" → UI render
    // request: "execute" → 실제 실행
}
```

UI에서 task 추가 시 plugin name 보임. Pipeline에서 사용.

## 2. SCM Plugin

git/svn 외 자체 SCM.
- `latest-revision`: 최신 commit 반환.
- `checkout`: 코드 가져옴.

## 3. Notification Plugin

```java
if (request.requestName().equals("notifications-interested-in")) {
    return ok("[\"stage-status\"]");
}
if (request.requestName().equals("stage-status")) {
    // pipeline data parse + Slack send
}
```

stage status change에 trigger.

## 4. Secret Plugin

```java
if (request.requestName().equals("go.cd.secrets.lookup")) {
    // {{SECRET:[my-config][key]}} resolve
    return ok(secretValueJson);
}
```

Vault, AWS SM 등 통합.

## 5. Elastic Agent Plugin

- `should-assign-work`: 이 agent에 작업 할당해야 하나?
- `create-agent`: 새 agent (Pod/container) 생성.
- `should-cancel-pending-task`: 큐 정리.
- `agent-status-report`: UI 표시 정보.

Kubernetes plugin 참고 (open source).

## 6. Configuration Plugin

YAML/JSON/Groovy → pipeline 변환.

```java
if (request.requestName().equals("parse-content")) {
    // YAML parse → JSON output
}
```

자체 DSL 만들 수 있음.

## 7. 자가평가
### Q1. Task plugin request? **get-view, execute**. 정답 1.
### Q2. Notification trigger? **stage-status change**. 정답 1.
### Q3. Secret plugin? **{{SECRET:...}} resolve**. 정답 1.
### Q4. Elastic Agent 4 method? **should-assign / create / cancel / status**. 정답 1.
### Q5. Configuration plugin? **DSL parse**. 정답 1.
