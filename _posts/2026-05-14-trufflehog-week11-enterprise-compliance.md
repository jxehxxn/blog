---
layout: post
title: "TruffleHog Week 11: 엔터프라이즈 거버넌스 — 컴플라이언스 매핑과 임원 메트릭"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops compliance
---

## 학습 목표

- 시크릿 스캐닝이 SOC 2 / PCI-DSS / HIPAA / ISO 27001의 어느 통제와 매핑되는지 안다.
- 감사 증빙(audit evidence)을 자동 생성·보존하는 파이프라인을 설계한다.
- 임원·이사회 보고용 메트릭 보드를 만든다.
- TruffleHog Enterprise/Cloud의 가치 명제를 평가한다.

## 1. 컴플라이언스 매핑 (개념)

각 규제는 "비밀번호·키 관리"를 직접 언급합니다. 상세 매핑은 [보충 4: 컴플라이언스 매핑] 포스트.

- **SOC 2 CC6.1**: Logical access — 자격 증명 보호.
- **PCI-DSS 3.5 / 8.2**: 암호 키 관리, 인증 자격 증명 보호.
- **HIPAA §164.312(a)**: Access control — 시크릿 노출은 PHI 접근 위험.
- **ISO 27001 A.9.4.3**: Password management system.
- **NIST 800-53 IA-5**: Authenticator management.

각 항목마다 감사인은 묻습니다: "그 통제가 실제로 동작한다는 증거는?"

## 2. 자동 증빙 파이프라인

```
[ trufflehog --json ] → [ S3 (WORM bucket, retention 7y) ]
                       ↓
                   [ Glue ETL → Parquet ]
                       ↓
                   [ Athena view: "evidence_v1" ]
                       ↓
                   [ Auditor-readable dashboard ]
```

WORM (Write Once Read Many) 또는 S3 Object Lock로 무결성 보장. 매주 SHA256 해시를 별도 ledger에 적재해 변경 불가능성 증명.

## 3. 보존 기간

| 규제 | 권장 보존 |
|------|----------|
| SOC 2 | 1년 (감사 주기) |
| PCI-DSS | 1년 (Req 10.7) |
| HIPAA | 6년 |
| ISO 27001 | 정책 정의 (보통 3년) |

빅테크 표준: **모든 보안 로그 7년 보존**으로 통일 (가장 긴 규제에 맞춤).

## 4. 임원 메트릭 보드

이사회·CEO 슬라이드 1장에 들어가는 메트릭 4종:

1. **Coverage**: 스캔 대상 리포 / 전체 리포 (% 표시).
2. **Verified incidents (월간)**: 진짜 누출 발생 건수.
3. **MTTR (revoke)**: 평균 시간 (목표 < 1h).
4. **Backlog**: baseline 안에 남아 있는 미처리 항목 (목표 → 0).

추가로 트렌드:
- Detector별 verified count의 6개월 추세.
- 사고 유형별 root cause 분포.

## 5. 운영 OKR 예시 (분기별)

- O: "시크릿 누출로 인한 비즈니스 임팩트 0"
- KR1: Coverage 100% 유지.
- KR2: MTTR-revoke < 30분.
- KR3: Verified incidents → 전 분기 대비 25% 감소.
- KR4: pre-commit 활성화 사내 리포 비율 ≥ 95%.

## 6. TruffleHog Enterprise / Cloud

오픈소스만으로도 빅테크가 잘 운영할 수 있지만, Enterprise 제품은 다음을 제공:

- 중앙 dashboard, 다중 조직 통합.
- SaaS 형태로 운영 부담 감소.
- 사전 통합된 detector 카탈로그 + 정기 업데이트.
- 통제·승인 UI (allowlist 워크플로 등).

**평가 기준** (구매 결정 시):
- 기존 SIEM·SOAR 통합 비용 절감 vs 라이선스 비용.
- 멀티테넌시 요구가 있는지 (자회사 다수).
- 컴플라이언스 인증(SOC 2 Type II 등) 필요성.

## 7. 빅테크 감사 사례 — SOC 2 Type II 통과 시 묻는 질문 5선

1. "시크릿 스캐닝 도구의 결과는 어디에 보존됩니까?" → S3 WORM, 7년.
2. "스캔 누락 가능한 리포가 있습니까?" → Coverage 100% 보고, 신규 리포 자동 등록 훅 보여줌.
3. "False Positive 분류 절차는?" → allowlist PR 워크플로 보여줌.
4. "검출 시 대응 SLA는?" → MTTR-revoke < 1h, 사례 ticket 제출.
5. "권한이 분리되어 있습니까?" → TruffleHog 토큰의 minimum permission 정책 보여줌.

## 8. 실습 과제

1. 위 4종 임원 메트릭을 BigQuery/Athena SQL로 직접 작성.
2. SOC 2 CC6.1 매핑 표 자체 작성 (도구 → 통제 → 증빙 → 보존).
3. 모의 감사 질문 5선에 대한 답변 슬라이드 1장(Markdown) 작성.

## 9. 자가평가 퀴즈

### Q1. SOC 2 CC6.1의 핵심 요구는?
1. Logical access — 자격 증명 보호.
2. 비즈니스 연속성.
3. 데이터 분류.
4. 백업.

**정답:** 1번.

### Q2. WORM 스토리지가 필요한 이유는?
1. 증빙의 무결성 — 사후 변조 방지.
2. 비용 절감.
3. 속도.
4. 무료.

**정답:** 1번.

### Q3. 임원 메트릭에 detector별 raw count를 직접 넣으면?
1. 직관적이라 좋다.
2. 너무 기술적이라 의사결정에 도움이 안 됨 — 비즈니스 임팩트 지표로 변환 필요.
3. 의미 없음.
4. 정확하다.

**정답:** 2번.

### Q4. MTTR-revoke 30분 목표가 의미하는 것은?
1. 알람부터 토큰 회수까지 평균 30분 이내.
2. 코드 리뷰 시간.
3. CI 빌드 시간.
4. 의미 없음.

**정답:** 1번.

### Q5. TruffleHog Enterprise를 도입할 때 1순위 고려사항은?
1. UI 디자인.
2. 기존 SIEM·SOAR 통합 비용·운영 부담 절감과 라이선스 비용의 균형.
3. 색상 테마.
4. CLI 옵션 다양성.

**정답:** 2번.

## 10. 다음 주차

[Week 12: 캡스톤]에서는 지금까지 배운 모든 것을 한 시뮬레이션으로 묶습니다. 채점 루브릭·모범답안은 별도 캡스톤 포스트에서.
