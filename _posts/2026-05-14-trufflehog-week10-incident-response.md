---
layout: post
title: "TruffleHog Week 10: 시크릿 누출 인시던트 대응 — NIST 800-61 매핑 + 60분 플레이북"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops incident-response
---

## 학습 목표

- NIST SP 800-61 인시던트 라이프사이클을 시크릿 누출 맥락에 매핑한다.
- "T+0 ~ T+60분" 시간순 플레이북을 외운다.
- 회수(revoke), 회전(rotate), 무력화(invalidate)를 구분한다.
- 포스트모템 템플릿과 비난-없는 문화(blameless culture)를 적용한다.

## 1. NIST 800-61 라이프사이클

```
Preparation → Detection & Analysis → Containment, Eradication & Recovery → Post-Incident
```

시크릿 누출 맥락 매핑:

1. **Preparation**: 도구·플레이북·on-call·access 권한 사전 정비.
2. **Detection**: TruffleHog verified 알람.
3. **Analysis**: ExtraData로 범위 파악, trace_id로 timeline 구성.
4. **Containment**: 토큰 즉시 회수, 관련 세션 종료.
5. **Eradication**: 히스토리 정리, 새 키 발급, 모든 사용처 재배포.
6. **Recovery**: 정상 동작 확인 + 모니터링 강화.
7. **Post-Incident**: 포스트모템, 재발 방지 액션.

## 2. T+0 ~ T+60 플레이북 (Big Tech 표준)

### T+0 ~ T+5 — 감지·확인
- TruffleHog 알람 수신 (Splunk Notable / PagerDuty).
- ExtraData로 detector·account·arn 확인.
- **거짓 양성 가능성 90초 안에 판단**: allowlist 매칭? baseline? 검증된 verified?
- 거짓이면 ticket close + reason. 진짜면 → T+5 단계.

### T+5 ~ T+15 — 격리 (Containment)
- **회수(revoke) 우선**: 토큰을 즉시 폐기.
  - AWS: `aws iam delete-access-key`
  - GitHub PAT: API로 revoke
  - GCP SA key: `gcloud iam service-accounts keys delete`
- 세션 종료: 해당 자격 증명으로 발급된 세션·STS 토큰 무효화.
- Vault dynamic이면 lease revoke (8주차).

### T+15 ~ T+30 — 분석 (Analysis)
- CloudTrail/Cloud Audit Logs로 **누가 그 토큰을 썼는가** 시간순 조회.
- 비정상 호출 (다른 region, 비정상 시각, 미사용 API) 식별.
- 데이터 유출/변경/생성 흔적 확인.

### T+30 ~ T+45 — 박멸 (Eradication)
- 히스토리에서 토큰 제거 (4주차 BFG/filter-repo).
- 신규 키 발급 + 사용처(서비스) 모두 재배포.
- 동일 패턴의 다른 토큰이 다른 곳에 또 있는지 광역 재스캔.

### T+45 ~ T+60 — 회복 + 통지
- 정상 동작 모니터링 확인.
- 영향받은 사용자/팀에 내부 공지.
- 외부 통지 의무(GDPR, HIPAA) 평가 — 법무팀 연결.

## 3. revoke vs rotate vs invalidate — 정확한 용어

- **Revoke**: 자격 증명 자체를 폐기. 그 토큰으로 더는 인증 불가.
- **Rotate**: 같은 principal에 새 자격 증명을 발급 + 옛 것 폐기. 서비스 무중단 목적.
- **Invalidate**: 발급된 세션·캐시·토큰을 무효화 (signing key 회전 시).

**3개 모두 다른 작업**입니다. 잘못 쓰면 사고가 커집니다.

## 4. 시뮬레이션 시나리오 — "T+0의 황금 5분"

**상황:** 새벽 3시, Splunk에서 TruffleHog Verified AWS key 알람. ExtraData.account = `595918472158`, arn = `arn:aws:iam::...:user/deploy-bot`.

**올바른 행동 시퀀스:**

1. (T+30s) PagerDuty 응답.
2. (T+90s) CMDB로 `deploy-bot` 책임 팀 확인 → `infra-team`.
3. (T+2m) Vault에 lease가 있다면 즉시 revoke. 없으면 AWS IAM 콘솔 → access key 비활성화.
4. (T+3m) STS GetCallerIdentity 호출이 차단되는지 검증.
5. (T+4m) CloudTrail 콘솔에서 마지막 24시간 deploy-bot 호출 검색.
6. (T+5m) infra-team Slack에 trace_id와 함께 사실 통보 (비난 X).

**잘못된 행동:**
- "일단 누가 commit 했는지 비난"하는 Slack 메시지 → 비난 문화 → 차후 신고 위축.
- 키만 회전하고 누출 원인 분석 생략.
- 통지 의무 평가를 누락.

## 5. 포스트모템 템플릿 (Blameless)

```markdown
# Incident postmortem: <trace_id>

## Summary (3 lines)
- 무엇이 누출됐는가
- 영향 범위
- 어떻게 해결했는가

## Timeline
- T+0: ...
- T+5: ...
- ...

## Root cause
- 직접 원인
- 기여 요인 (시스템·프로세스·문화)

## What went well
- ...

## What can improve
- ...

## Action items (owner, due date)
- ...
```

빅테크 원칙:
- **사람 이름 대신 시스템·프로세스에 책임**.
- **모든 액션 아이템에 책임자 + 기한 + 추적 ticket**.
- 모든 포스트모템은 사내 공개 (학습 자산).

## 6. 빅테크 사례 — GitHub의 token rotation 자동화

GitHub 자체가 사고를 겪었을 때, 사후 모든 OAuth/PAT 토큰을 자동 회전했습니다. 핵심은 **회전이 사고 시점에 한 번에 일어날 수 있는 인프라가 평소에 준비되어 있었다는 것**. 이게 Preparation의 진짜 의미입니다.

## 7. 실습 과제 (테이블탑 시뮬레이션)

1. 위 시나리오를 동료 1명과 30분 테이블탑으로 진행.
2. 한 명은 SOC 분석가, 한 명은 infra-team 책임자.
3. T+0부터 T+60까지 결정·조치를 기록.
4. 60분 후 포스트모템 1쪽 작성.
5. 누락된 액션·blameless 위반 횟수 자기 평가.

## 8. 자가평가 퀴즈

### Q1. revoke와 rotate의 차이?
1. 같은 말.
2. revoke는 폐기, rotate는 폐기+재발급(무중단 목적).
3. revoke는 무중단, rotate는 중단.
4. 둘 다 의미 없음.

**정답:** 2번.

### Q2. 시크릿 누출 인시던트의 첫 번째 조치는?
1. 비난 Slack.
2. 토큰 회수.
3. 포스트모템 작성.
4. 코드 리뷰.

**정답:** 2번.

### Q3. Blameless culture의 이유는?
1. 신고가 위축되지 않게 → 진짜 사고를 빨리 알 수 있게.
2. 책임자가 없어도 됨.
3. 법적 보호.
4. 무의미.

**정답:** 1번. 비난 문화는 사고를 숨기게 만듭니다.

### Q4. T+5분 이내에 끝내야 하는 것은?
1. 토큰 회수 + 책임 팀 식별.
2. 전체 포스트모템.
3. 외부 통지.
4. 코드 변경 PR.

**정답:** 1번.

### Q5. 통지 의무 평가가 별도로 필요한 이유는?
1. GDPR·HIPAA 등 규제 통지 기한이 짧다(보통 72시간).
2. 마케팅.
3. 비용 절감.
4. 의미 없음.

**정답:** 1번.

## 9. 다음 주차

[Week 11: 엔터프라이즈/거버넌스]에서는 SOC 2 / PCI-DSS / HIPAA에 시크릿 스캐닝을 매핑하고, 임원·감사인에게 보여줄 메트릭 보드를 설계합니다.
