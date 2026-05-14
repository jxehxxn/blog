---
layout: post
title: "TruffleHog Capstone: 채점 루브릭과 3종 모범답안 (스타트업/스케일업/엔터프라이즈)"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops capstone
---

본 포스트는 [Week 12 캡스톤]의 평가 기준과 모범답안 3종을 상세히 풉니다.

## 1. 채점 루브릭 — 총 100점

### 기준 1: 설계 적정성 (40점)

| 항목 | 점수 | 평가 기준 |
|------|------|----------|
| 4계층 방어선 | 10 | IDE/pre-commit/CI/org scan을 모두 포함하고 각각의 역할이 명시 |
| 사내 detector + verifier | 10 | regex 구현 + verifier (캐시·rate limit·회로차단 패턴 명시) |
| 데이터 파이프라인 | 10 | idempotency key + trace_id 구현, 결과 저장 + 시각화 |
| 컴플라이언스 매핑 | 10 | SOC 2 CC6.1 매핑 표 + evidence 보존 (WORM, 7년) |

각 항목은 0/3/6/10 단계로 채점. 0: 누락, 3: 개념만, 6: 부분 구현, 10: 운영 가능.

### 기준 2: 운영 완성도 (30점)

| 항목 | 점수 | 평가 기준 |
|------|------|----------|
| CI best practice | 10 | fetch-depth, --no-update, --fail, --concurrency, artifact 90+ days |
| FP 튜닝 | 10 | allowlist 5계명 적용, baseline diff CI 코드 포함 |
| 메트릭 정의 | 10 | Coverage/Verified/MTTR/Backlog 4종 측정 가능 SQL |

### 기준 3: 인시던트 대응 (30점)

| 항목 | 점수 | 평가 기준 |
|------|------|----------|
| T+0~T+60 단계 | 10 | 5단계 모두, 각 단계별 actor·도구·SLA 명시 |
| revoke/rotate/invalidate | 10 | 정확한 용어 사용 + AWS·GitHub·Vault 별 예시 |
| Blameless 포스트모템 | 10 | 템플릿 + 통지 의무(GDPR/HIPAA) 평가 분기 포함 |

## 2. 모범답안 A — 스타트업 (50명 이하, 리포 20개)

### 설계
- **4계층**: pre-commit (필수) + GitHub Actions PR CI (verified=차단) + 주간 cron으로 전 org 스캔. pre-receive hook은 도입하지 않음(비용 ROI 낮음).
- **detector**: 사내 토큰이 단순하므로 YAML 정의로 시작. verifier는 없음. 6개월 후 Go detector로 전환 검토.
- **파이프라인**: GitHub Actions → JSON artifact → S3 (1년 retention, 비용 절감).
- **컴플라이언스**: SOC 2 미준비. ISO 27001만 우선 (스타트업 표준).

### 운영
- CI: `fetch-depth: 0`, `--no-update`, `--only-verified --fail`. 단순.
- FP: 처음 1주는 모든 알람을 사람이 직접 검토 → baseline 작성 → 그 후 baseline diff.
- 메트릭: Coverage, Verified count, MTTR (수동 측정 OK).

### 인시던트
- T+0: Slack `#sec-alerts`에 알람.
- T+5: 책임 엔지니어가 GitHub 콘솔에서 토큰 revoke.
- T+15: CloudTrail 콘솔 수동 확인.
- T+60: Slack 공지 + 1쪽 짜리 포스트모템 (Notion).

### 평가 시 강점/약점
- 강점: ROI 명확, 신속한 도입.
- 약점: pre-receive hook 부재, verifier 없음. 사고가 잦아지면 한계.

## 3. 모범답안 B — 스케일업 (200명, 리포 200개, 시리즈 C)

### 설계
- **4계층 완전**: IDE 플러그인까지 0층 도입. pre-receive hook은 GitLab self-hosted 한정 (감사 대응 차원).
- **detector**: Go 코드로 사내 fork 운영. verifier에 캐시 + rate limiter 적용. 회로차단은 추후.
- **파이프라인**: TruffleHog → SQS → Lambda → S3 + DynamoDB(트랙용) → Athena → Looker dashboard.
- **컴플라이언스**: SOC 2 Type II 진행 중. CC6.1, CC6.7, CC7.2 매핑 표 완비.

### 운영
- CI: GitHub Actions + GitLab CI 양쪽. fetch-depth + --no-update + --fail + artifact 90일.
- FP: allowlist 5계명 + baseline 도입. detector confidence 등급 (HIGH/MEDIUM/LOW).
- 메트릭: 4종 모두 SQL로 자동 계산.

```sql
-- Coverage
SELECT
  COUNT(DISTINCT repo) FILTER (WHERE scanned_within_7d) * 1.0
  / COUNT(DISTINCT repo) AS coverage
FROM repo_catalog;
```

### 인시던트
- T+0: PagerDuty(SOC 1차) + Slack on-call channel.
- T+5: revoke (Vault → AWS IAM → GitHub PAT 순서).
- T+15: CloudTrail SQL 자동 조회.
- T+30: 영향 분석 자동화 (lambda 결과).
- T+60: 포스트모템 템플릿(Linear ticket) + 외부 통지 평가 (법무 1차).

### 평가 시 강점/약점
- 강점: 자동화 완성도, 메트릭 운영.
- 약점: pre-receive hook 부재로 직접 push에 빈틈. 회로차단 미도입.

## 4. 모범답안 C — 엔터프라이즈 (5,000명, 리포 5,000+, 멀티 클라우드)

### 설계
- **4계층 + 0층**: IDE 플러그인 강제, pre-commit auto-install, GitHub Enterprise + GitLab self-hosted에 pre-receive hook 모두 배포.
- **detector**: Go 사내 fork, 내부 token 형식 10+ 개 지원. 모든 verifier에 캐시·rate limit·회로차단 적용. **dry-run 모드**도 운영 (사내 API 장애 대비).
- **파이프라인**: TruffleHog → Kafka → Spark Streaming → S3 (WORM, 7년) + BigQuery → Looker. SOAR(Tines) 연동 자동.
- **컴플라이언스**: SOC 2 Type II + PCI-DSS + HIPAA + ISO 27001 모두 매핑 완비.

### 운영
- CI: 모든 GitHub Org에 GitHub Action template, 모든 GitLab Project에 CI template. **위반 시 머지 금지 강제** (Branch protection ruleset).
- FP: allowlist + baseline + confidence + detector ML scoring (자체 학습 분류기 보조).
- 메트릭: 4종 + 트렌드 + detector별 ROC curve. 이사회 슬라이드 1장 자동 생성.

### 인시던트
- T+0: Splunk Enterprise Security Notable Event + PagerDuty.
- T+5: SOAR 자동 워크플로 (revoke + 영향 분석 + Slack 통보) 모두 자동.
- T+15: 인간 in-the-loop 분석 시작.
- T+60: 자동 포스트모템 초안 생성 → 인간 보완.
- 통지 의무 자동 평가: GDPR/HIPAA 트리거 시 법무팀 자동 페이지.

### 평가 시 강점/약점
- 강점: 거의 모든 영역 자동화, 운영 완성도 최고.
- 약점: 운영 인프라 비용 크고 변경 관리 부담. 도구의 도구 (도구 자체의 보안 통제)가 별도 과제.

## 5. 채점 시뮬레이션 — 예시 제출본 평가

가상의 제출본을 보고 점수를 매기는 연습:

**제출 요약:**
- pre-commit + GitHub Actions만 (org scan 누락)
- 사내 detector YAML로 작성, verifier 없음
- 결과 S3 적재 + Looker
- 인시던트 플레이북 T+15까지만 정의

채점:
- 기준 1: 4계층(6점, org scan 누락) + detector(3, verifier 누락) + 파이프라인(6, idempotency 명시 X) + 컴플라이언스(0, 매핑 없음) = **15/40**
- 기준 2: CI(7) + FP(3, allowlist만) + 메트릭(3) = **13/30**
- 기준 3: 단계(5, T+15까지) + 용어(7) + 포스트모템(0) = **12/30**
- 합계: **40/100** — fail. 재제출 필요.

피드백: "org scan과 컴플라이언스 매핑이 누락. 인시던트 T+30 이후 정의 필요."

## 6. 마무리

이 채점 루브릭은 빅테크 보안팀의 실제 면접·재능 평가 기준에 가깝습니다. 100점을 받는 것보다, **약점이 어디인지 정확히 진단하는 능력**이 더 중요합니다.
