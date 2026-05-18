---
layout: post
title: "GoCD 보충 1: Continuous Delivery 책 핵심"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd cd-book supplement
---

GoCD의 모태 책 핵심 정리.

## 1. 책 정보
- 저자: Jez Humble, David Farley.
- 출간: 2010.
- 출판: Addison-Wesley.

## 2. 핵심 메시지
**"매 commit이 production 배포 가능한 상태여야 한다."**

CI(빌드만)와 CD(배포 가능)의 명확한 구분.

## 3. 8 원칙

1. SW 출시 process는 반복 가능 + 신뢰 가능해야.
2. 거의 모든 것을 자동화.
3. 어렵고 고통스러운 것은 자주 한다 (작게).
4. 품질을 처음부터 (shift-left).
5. "Done" 의 정의를 명확히 (production까지).
6. 모두가 모든 것에 책임.
7. 지속 개선.
8. 변경의 cost를 줄임.

## 4. Deployment Pipeline

책의 시그니처 아이디어. 각 단계가 명확한 의도:
- Commit stage (build + unit test).
- Acceptance test stage.
- Capacity test stage.
- Manual UAT stage.
- Production deploy stage.

GoCD가 정확히 이 모델 구현.

## 5. Configuration Management

- 모든 것을 git에 (manifest, config, 환경).
- DB schema 포함.

## 6. Feedback Loop

- Fast (수분 내).
- Reliable (오탐 X).
- Comprehensive (의미 있는).

## 7. Anti-pattern

- Manual deploy.
- Production-only config.
- Lengthy release.

## 8. 영향
- DevOps 운동의 출발 기여.
- 모든 modern CI/CD 도구의 기초.

## 9. 결론

이 책 안 읽고 GoCD/CD 도구를 사용하는 것은 도구만 보고 철학 모르는 것. 강력 추천.
