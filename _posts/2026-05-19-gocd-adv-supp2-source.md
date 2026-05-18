---
layout: post
title: "GoCD 심화 보충 2: Source Code Internals (Java)"
date: 2026-05-19 10:00:00 +0900
categories: cicd gocd source supplement
---

## Repo
https://github.com/gocd/gocd

## 구조
- `server/`: main server (Spring + Jetty).
- `agent/`: agent.
- `common/`: 공유.
- `domain/`: business logic.
- `plugin-infra/`: plugin loading.
- `api/`: REST API.
- `webapp/`: frontend (React).

## Build
```bash
./gradlew build
```

## Test
JUnit + Cucumber e2e.

## Contribution

1. Fork + branch.
2. Issue claim.
3. PR + sign DCO.
4. Review (2+).
5. Merge.

## Community
- Slack: gocd-dev.
- ThoughtWorks office hour.

## 결론
큰 codebase (수십만 line). good-first-issue부터.
