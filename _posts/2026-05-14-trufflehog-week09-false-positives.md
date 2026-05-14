---
layout: post
title: "TruffleHog Week 9: False Positive 튜닝 — allowlist, baseline, confidence 운영"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops
---

## 학습 목표

- True/False Positive/Negative 4분면을 시크릿 스캐닝 맥락으로 이해한다.
- allowlist를 안전하게 운영하는 방법을 안다.
- baseline 파일 기반 점진 도입 전략을 적용한다.
- detector별 confidence 모델을 운영 메트릭과 연결한다.

## 1. 4분면 — 시크릿 스캐닝의 "오답 매트릭스"

|              | 실제 시크릿 (positive) | 실제 일반 문자열 (negative) |
|--------------|-----------------------|-----------------------------|
| 알람 발생    | True Positive (잡음)  | **False Positive (소음)**   |
| 알람 미발생  | **False Negative (누락)** | True Negative (조용함)  |

빅테크에서 가장 두려운 건 **False Negative** — 도구가 못 잡은 진짜 시크릿이 살아 있는 경우. 하지만 운영 피로를 만드는 건 **False Positive** — 진짜가 아닌데 매번 알람.

목표: FP 비율을 5% 이하로 유지하면서 FN을 0에 가깝게.

## 2. False Positive의 주요 원인

1. **테스트 데이터**: 일부러 만든 "fake" 토큰이 진짜 패턴과 일치.
2. **하드코딩 예시**: 문서·블로그 안에 예시용 토큰.
3. **충돌하는 형식**: 일반 base64가 secret access key 패턴과 닮음.
4. **레거시 keystore**: 이미 폐기되어 동작하지 않는 토큰.

## 3. Allowlist — 안전한 운영 5계명

1. **항상 코드 안에 두고 PR 리뷰**. 절대 운영자가 콘솔에서 누르지 말 것.
2. **항목마다 만료일** (예: 90일). 만료 시 자동 재검토.
3. **사유 필수 기재** (`reason: "test fixture for unit tests"`).
4. **owner 필드**: 책임자 명시.
5. **wildcard 금지**. 정확한 경로/해시로 좁히기.

TruffleHog `--exclude-paths` 예시 + 사내 allowlist 포맷 제안:

```yaml
# .trufflehog-allowlist.yml
version: 1
entries:
  - id: 2024-01-15-stripe-test-fixture
    path: tests/fixtures/stripe_webhook.json
    detector: Stripe
    hash: sha256:f3b9...
    reason: "Stripe webhook signature test fixture, public test key"
    owner: payments-team
    expires: 2024-04-15
```

CI 통합:

```bash
trufflehog filesystem . --exclude-paths .trufflehog-exclude.txt \
  --only-verified \
  | python scripts/apply-allowlist.py .trufflehog-allowlist.yml
```

## 4. Baseline — 점진 도입의 핵심

기존 monorepo에 처음 도구를 도입하면 수백 개 알람이 쏟아집니다. baseline 패턴:

1. 최초 1회 풀스캔 → 모든 결과를 baseline 파일로 저장.
2. CI는 **baseline에 없는 새 알람만** 차단.
3. baseline 내 항목은 별도 backlog로 점진 해결.
4. baseline 자체가 정기적으로 줄어들도록 OKR 설정.

```bash
# 최초 baseline
trufflehog filesystem . --only-verified --json > baseline.json

# CI에서 비교
trufflehog filesystem . --only-verified --json > current.json
python scripts/diff.py baseline.json current.json --fail-on-new
```

## 5. Detector별 Confidence 모델

같은 도구 안에서도 detector의 신뢰도가 다릅니다.

- AWS, Stripe, GitHub PAT: verifier 가 있고 정확 → confidence HIGH.
- 일반 password 정규식: regex만, FP 폭증 → confidence LOW.

빅테크 운영:
- HIGH: PR 차단 + 즉시 페이지.
- MEDIUM: PR 코멘트 + 일간 review.
- LOW: 주간 review만.

confidence는 detector 메타데이터로 가지고 있다가, 결과에 태그합니다.

## 6. 메트릭과 SLO

- Verified TP rate: 검증된 결과 중 실제 누출 비율 (목표 ≥ 95%).
- FP rate (운영자 평가): 알람 중 실제로 무시할 수 있는 비율 (목표 ≤ 5%).
- MTTR (revoke): 누출 시작 ~ 토큰 회전 완료 (목표 < 1시간).
- Coverage: 전체 리포 중 스캔되는 비율 (목표 100%).

이 메트릭을 11주차에서 거버넌스 보고로 변환합니다.

## 7. 실습 과제

1. 본인 리포에 실제 같은 패턴(테스트 fixture 포함)을 5종 추가.
2. 스캔 후 어떤 것이 TP/FP인지 직접 분류.
3. allowlist 파일을 만들어 FP만 무시되도록 구성.
4. CI에서 baseline diff 로직 1쪽짜리로 작성.
5. 한 detector를 골라 confidence 등급을 부여하고 정당화 단락 작성.

## 8. 자가평가 퀴즈

### Q1. FP가 너무 많을 때의 1차 위험은?
1. 시스템 성능.
2. 알람 피로(alert fatigue)로 진짜 알람도 무시됨.
3. 디스크 용량.
4. CI 시간.

**정답:** 2번. 운영 보안의 고전적 함정.

### Q2. allowlist에 wildcard를 허용하지 않는 이유는?
1. 정확하지 않으면 진짜 누출이 묻힘.
2. 성능 문제.
3. 가독성.
4. CI가 안 돌아감.

**정답:** 1번.

### Q3. baseline 패턴의 핵심 가치는?
1. 점진 도입 — 도구를 켜자마자 전 PR을 차단하지 않음.
2. 데이터 압축.
3. detector 비활성화.
4. 토큰 회전.

**정답:** 1번.

### Q4. detector confidence가 의미하는 것은?
1. 알람이 진짜 시크릿일 확률에 대한 사전 신뢰도.
2. 도구 성능.
3. 자동 회전 가능성.
4. 의미 없음.

**정답:** 1번.

### Q5. MTTR-revoke의 목표가 짧을수록 좋은 이유는?
1. 노출 시간이 짧을수록 비즈니스 임팩트가 작다.
2. 도구가 빨라 보인다.
3. 비용 절감.
4. 메트릭 점수 향상.

**정답:** 1번.

## 9. 다음 주차

[Week 10: 인시던트 대응]에서는 NIST 800-61 5단계(Prepare/Detect/Contain/Eradicate/Recover) 매핑을 통해 실제 시크릿 누출 발생 시 시간순 플레이북을 다룹니다.
