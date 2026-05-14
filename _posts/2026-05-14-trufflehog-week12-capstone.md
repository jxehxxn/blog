---
layout: post
title: "TruffleHog Week 12: 캡스톤 프로젝트 — Big Tech 시뮬레이션 풀스택"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops capstone
---

## 캡스톤 개요

12주간 배운 모든 컴포넌트(detector·verifier·CI·SIEM·SOAR·거버넌스)를 묶어 **가상의 빅테크 환경에 시크릿 스캐닝 프로그램을 처음부터 구축**하는 프로젝트입니다.

## 1. 시나리오

당신은 가상 회사 **MegaCorp**에 새로 들어온 Senior DevSecOps 엔지니어입니다. 회사 상황:

- GitHub Enterprise Cloud, 1,200개 리포, 200명 엔지니어.
- 사내 토큰 형식: `mc_<env>_<32hex>` (env = dev|stage|prod).
- AWS 계정 8개, GCP 프로젝트 12개, HashiCorp Vault 운영 중.
- SOC 2 Type II 인증 진행 중 (3개월 후 감사).
- 지난 6개월 동안 시크릿 누출 인시던트 3건 (모두 사후 발견).

CEO가 요청: **"3개월 안에 누출 0 만들고, 감사 통과시켜"**.

## 2. 산출물

다음 5종을 모두 제출합니다.

1. **아키텍처 다이어그램** (PNG/SVG): 4계층 방어선 + 데이터 파이프라인.
2. **detector 코드** (Go): 사내 토큰 `mc_<env>_<32hex>`용 detector + verifier.
3. **CI 통합 YAML**: GitHub Actions + pre-commit.
4. **인시던트 플레이북** (Markdown): T+0~T+60분 + 포스트모템 템플릿.
5. **거버넌스 1쪽**: 임원 메트릭 보드 + SOC 2 매핑 표.

## 3. 평가 기준 (Grading Rubric) — 3종

채점은 100점 만점, 각 영역 점수 합산.

### 기준 1: 설계 적정성 (40점)
- [10pt] 4계층 방어선이 모두 포함되었는가
- [10pt] 사내 토큰 detector + verifier가 idempotency/caching/rate limit/회로차단을 갖추는가
- [10pt] 데이터 파이프라인이 idempotency key + trace_id를 구현하는가
- [10pt] SOC 2 CC6.1 매핑이 명시적이고 evidence 보존 방안이 있는가

### 기준 2: 운영 완성도 (30점)
- [10pt] CI가 fetch-depth, --no-update, --fail 등 운영 best practice를 반영
- [10pt] FP 튜닝(allowlist, baseline) 전략이 문서화
- [10pt] 메트릭 4종(Coverage, Verified, MTTR, Backlog)이 측정 가능한 정의로 작성

### 기준 3: 인시던트 대응 시나리오 (30점)
- [10pt] T+0~T+60 단계별 행동이 모두 정의
- [10pt] revoke/rotate/invalidate를 정확한 용어로 구분 사용
- [10pt] Blameless 포스트모템 템플릿 + 통지 의무 평가 절차 포함

## 4. 자세한 채점 루브릭 + 3종 모범답안

별도 포스트 [TruffleHog Capstone: 채점 루브릭과 3종 모범답안]에서 다룹니다 (스타트업/스케일업/엔터프라이즈 각각).

## 5. 마무리

이 캡스톤을 완성하면 여러분은 빅테크 환경에서 **시크릿 스캐닝 프로그램을 처음부터 끝까지 책임지고 운영**할 수 있는 수준에 도달한 것입니다. 12주 동안 수고하셨습니다.

후속 학습 추천:
- TruffleHog 본가 contributor 되기 (regex PR로 시작).
- 회사에서 실제 도입 → 메트릭 추적 → 분기 회고.
- 다른 secret-adjacent 영역(SBOM, SCA, container scanning)으로 확장.
