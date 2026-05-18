---
layout: post
title: "O11y Week 11: 인시던트 대응 + Blameless Postmortem"
date: 2026-05-18 18:00:00 +0900
categories: observability sre incident-response platform senior-series
---

## 학습 목표

- 인시던트 5단계 (detect / triage / mitigate / resolve / postmortem).
- ICS (Incident Command System) 적용.
- Blameless 문화의 본질.
- Action item 추적.

## 1. 비유 — "응급실 운영"

응급환자 입원 → 분류 (triage) → 응급 처치 (mitigate) → 본 치료 (resolve) → 사례 회의 (postmortem). 인시던트도 같은 구조.

## 2. 5 단계

### Detect
SLO burn rate alert / customer 신고 / monitoring spike.

### Triage
- 심각도 (SEV1/2/3/4) 결정.
- 영향 범위 추정.
- IC (Incident Commander) 지정.

### Mitigate
즉시 영향 차단. rollback / 트래픽 차단 / restart. 근본 해결 X, 일단 안정화.

### Resolve
근본 원인 수정.

### Postmortem
사후 회의 + 문서 + action item.

## 3. SEV 등급

- SEV1: 광범위 outage, 매출 영향. all-hands.
- SEV2: 중요 기능 영향. team mobilize.
- SEV3: 부분 영향. 평소 시간 내 대응.
- SEV4: 사소.

각 SEV별 escalation policy.

## 4. ICS (Incident Command System)

위기 응급 대응 군대식 체계.

- **Incident Commander (IC)**: 의사결정 단일 책임.
- **Communication Lead (CL)**: 외부/내부 소통.
- **Subject Matter Expert (SME)**: 기술 전문가.
- **Scribe**: 시간순 기록.

각 역할 명확. 사고 중 IC가 결정, 나머지는 IC 결정 따라 움직임.

## 5. Mitigation의 5 옵션

1. Rollback (가장 빠름, 가장 흔함).
2. Feature flag off.
3. 트래픽 차단 (사용자 격리).
4. Scale up (자원 부족 시).
5. Manual fix (마지막 옵션).

빅테크 표준: 1번을 90% 이상에서 채택. 그래서 GitOps + rollback 자동화가 핵심.

## 6. Communication

### 내부
- Slack `#incident-xxx` 전용 채널.
- 5분마다 update.
- ICS 역할 pin.

### 외부
- Statuspage / 트위터.
- 30분마다 update.
- 정확한 시간·범위·예상 복구.

## 7. Postmortem 템플릿

```markdown
# Postmortem: <date> - <title>

## Summary (3 lines)
- 무엇이 / 영향 / 어떻게 해결.

## Impact
- 영향받은 사용자 / 시간 / 매출.

## Timeline
- T+0: alert fire.
- T+5: IC 지정.
- T+10: rollback.
- T+15: 안정화.
- T+60: resolve.

## Root cause
- 직접 원인.
- 기여 요인 (시스템·프로세스·문화).

## What went well
- ...

## What can improve
- ...

## Action items
| Action | Owner | Due | Status |
|--------|-------|-----|--------|
| ... | @user | 2026-06-01 | open |
```

## 8. Blameless 문화

비유: 의료 사고를 의사 개인 잘못으로 돌리면 다음 사고를 숨기게 됨. 시스템 결함을 솔직히 말할 수 있어야 학습.

원칙:
- 사람 이름 + 비난 대신 "어떤 결정이 어떤 정보로 내려졌는지".
- 같은 상황에 다른 누구라도 같은 실수를 할 수 있는가?
- 만약 그렇다면 시스템·프로세스 결함.

## 9. Action Item 추적

postmortem 후 action item을 ticket으로 강제. 다음 분기 OKR에 포함.

추적 안 하면 같은 사고 반복. 빅테크 가장 흔한 실패.

## 10. Game Day / Chaos Engineering

평시에 의도적 실패 주입 (Chaos Mesh, Litmus, Gremlin).
- "이 서비스가 죽으면 어떻게 되나?"
- 대응 절차 검증.
- 새 oncall 훈련.

빅테크 분기 1회 game day가 표준.

## 11. 운영 함정 5선

1. IC 부재 → 의사결정 분산.
2. mitigation 건너뛰고 root cause 분석 시도 → 사고 장기화.
3. postmortem 작성만 하고 action item 추적 X.
4. Blame culture → 사고 은닉.
5. Game day 부재 → 실전이 첫 시도.

## 12. 빅테크 사례

### Google
SRE 책의 모태. 모든 incident postmortem 사내 공개.

### GitHub
일부 postmortem 공개 (사용자 신뢰 회복).

### Stripe
incident response를 product engineering 표준으로.

## 13. 실습 (테이블탑)

가상 시나리오: "결제 service의 p99 latency가 30초로 spike. 30분 후 SLO budget 50% 소진. IC로서 어떻게 대응?"

5단계로 시간순 응대 작성.

## 14. 자가평가 퀴즈

### Q1. 인시던트 5단계?
1. **Detect/Triage/Mitigate/Resolve/Postmortem**
2. 무관
3. UI
4. 4단계

**정답: 1.**

### Q2. ICS의 IC 역할?
1. **단일 의사결정 책임**
2. 기록
3. 외부 소통
4. SME

**정답: 1.**

### Q3. 빅테크 mitigation 90% 1번?
1. **Rollback**
2. Manual fix
3. Scale up
4. 무관

**정답: 1.**

### Q4. Blameless 문화의 본질?
1. **사고 학습 + 은닉 방지**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. Game Day의 가치?
1. **평시 대응 절차 검증 + 새 oncall 훈련**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 15. 다음 주차

[Week 12: 캡스톤]에서 모든 학습을 묶어 풀스택 o11y 플랫폼.
