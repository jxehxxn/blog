---
layout: post
title: "TruffleHog Week 8: 도구 연동 — Splunk SIEM, HashiCorp Vault, Tines SOAR"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops
---

## 학습 목표

- TruffleHog 결과를 Splunk HEC로 흘려보내는 파이프라인을 만든다.
- Vault에 동적 시크릿(dynamic secret)이 노출됐을 때 자동 회전 트리거를 설계한다.
- Tines/Demisto SOAR로 인시던트 워크플로를 자동화한다.
- 모든 통합에 공통되는 idempotency·trace_id 패턴을 안다.

## 1. 통합 아키텍처 한눈에

```
[ trufflehog --json ]
        ↓
[ stream processor (Lambda/CF) ]
        ↓
   ┌────────┬──────────┬──────────┐
   ↓        ↓          ↓          ↓
[Splunk] [Vault]   [Tines]   [Slack/Jira]
 (감지)  (회전)    (대응)    (알림)
```

핵심 디자인 원칙:
- TruffleHog는 **검출만** 한다. 회전·대응은 별도 시스템.
- 모든 메시지는 `trace_id`로 묶여 한 사건이 추적 가능.
- 같은 토큰의 중복 메시지는 idempotency key로 흡수.

## 2. Splunk HEC 연동

```bash
trufflehog git <repo> --only-verified --json | \
while IFS= read -r line; do
  curl -k -X POST \
    -H "Authorization: Splunk $HEC_TOKEN" \
    -d "{\"event\": $line, \"sourcetype\": \"trufflehog:v3\"}" \
    "https://splunk.mycorp.com:8088/services/collector/event"
done
```

Splunk에서 사용할 검색 예 (SPL):

```spl
index=security sourcetype="trufflehog:v3" Verified=true
| stats count by DetectorName, ExtraData.account
| sort -count
```

빅테크 패턴: **Splunk Notable Event**를 만들어 SOC 큐로. ES(Splunk Enterprise Security)면 correlation search.

## 3. HashiCorp Vault 회전 트리거

가정: AWS IAM 사용자가 Vault dynamic secret으로 관리되고, 누출 발생 시 즉시 폐기 + 재발급.

```python
import hvac

client = hvac.Client(url="https://vault.mycorp.com", token=VAULT_TOKEN)

def revoke_aws_lease(account_id, principal):
    # 1. 누출된 lease 찾기
    leases = client.sys.list_leases(prefix=f"aws/creds/{principal}")
    for lid in leases.get("data", {}).get("keys", []):
        client.sys.revoke_lease(lease_id=f"aws/creds/{principal}/{lid}")
        log(f"revoked {lid}")

    # 2. 후속 발급은 자동으로 다음 호출 시 새 키
```

흐름:
1. TruffleHog가 verified AWS 키 발견.
2. ExtraData.account, ExtraData.arn으로 principal 추출.
3. Vault API로 해당 principal의 모든 lease revoke.
4. CloudTrail 확인으로 즉시 차단 검증.

**중요**: Vault 외부에서 발급된 정적 키라면 IAM API로 직접 deactivate해야 합니다. (10주차)

## 4. Tines / Demisto SOAR

Tines 스토리(workflow) 예시 (개념):

```
[Webhook trigger]
    ↓
[Enrich: ExtraData에서 account/arn 파싱]
    ↓
[Lookup: 어떤 팀 책임인가? CMDB 조회]
    ↓
[Decision: account가 운영 환경?]
    ├─ Y → [Revoke immediately] → [Page on-call]
    └─ N → [Open Jira ticket, P2]
    ↓
[Comment on PR / Slack DM to author]
    ↓
[Track: ticket close 시까지 SLA 타이머]
```

`trufflehog --json`을 Tines Webhook으로 직접 흘리는 방법:

```bash
trufflehog git <repo> --only-verified --json | \
  curl -X POST -H "Content-Type: application/json" \
       -d @- "https://hooks.tines.com/$STORY_ID"
```

## 5. Idempotency — 같은 사건을 두 번 처리하지 않기

10분마다 같은 commit을 다시 스캔하면 같은 시크릿 매칭이 또 발생합니다. 이걸 또 회전·페이지하면 시스템이 산다.

해결: **idempotency key**.

```python
def idempotency_key(result):
    return sha256(
        f"{result['DetectorName']}|{result['Raw']}|{result['SourceMetadata']['Git']['commit']}"
    )
```

이 키를 Redis/Dynamo에 24시간 TTL로 저장하고, 이미 있으면 스킵.

## 6. Trace ID — 한 사건의 모든 흔적 묶기

```python
trace_id = uuid4()
splunk_event["trace_id"] = trace_id
vault_audit_tag = f"trufflehog-{trace_id}"
tines_input["trace_id"] = trace_id
jira_ticket.fields["customfield_trace"] = trace_id
```

10주차 인시던트 대응에서 이 trace_id로 timeline 재구성합니다.

## 7. 빅테크 현장 사례 — Stripe의 "Sorbet" 패턴

Stripe는 (공개 발표 기준) 시크릿 검출과 회전을 한 파이프라인으로 묶어 운영합니다. detector 알람 → 즉시 회전 시도 → 회전 성공이면 PR 머지 허용. 핵심은 **회전이 사후 처리가 아니라 검출 직후 0초 안에 일어나는 것**입니다. 이게 진짜 "shift-left + automate-right".

## 8. 실습 과제

1. 무료 Splunk Cloud Trial을 만들어 HEC 토큰을 발급.
2. TruffleHog 결과를 HEC로 보내는 1쪽짜리 스크립트 작성.
3. Splunk에서 verified만 보여주는 dashboard 1개 만들기.
4. (선택) Vault Dev 모드를 띄우고 회전 트리거 흐름을 mock 함수로 구현.
5. trace_id가 잘 propagate 되는지 검증.

## 9. 자가평가 퀴즈

### Q1. TruffleHog의 역할 경계는?
1. 검출만, 회전·대응은 별도 시스템.
2. 검출·회전·대응 모두.
3. 회전만.
4. 대응만.

**정답:** 1번. 책임 분리(Separation of Concerns).

### Q2. idempotency key가 없으면?
1. 동일 사건을 반복 처리해 시스템·SOC가 마비.
2. 의미 없음.
3. 정확도 향상.
4. 비용 절감.

**정답:** 1번.

### Q3. Vault dynamic secret을 쓰면 회전이 쉬운 이유는?
1. lease 단위로 즉시 폐기 가능.
2. 자동 백업.
3. 무료.
4. 모두 맞음.

**정답:** 1번.

### Q4. trace_id의 핵심 가치는?
1. 한 사건의 모든 시스템 로그를 묶어 timeline 재구성.
2. 비용 추적.
3. UI 식별자.
4. 의미 없음.

**정답:** 1번.

### Q5. SOAR가 하는 일은?
1. 자동 워크플로(트리거 → 조건 → 액션) 오케스트레이션.
2. 시크릿 저장.
3. CI 빌드.
4. 단순 알림.

**정답:** 1번.

## 10. 다음 주차

[Week 9: False Positive 튜닝]에서는 allowlist/baseline/confidence 모델을 다루며 detector별 FP 비율을 5% 이하로 만드는 운영 노하우를 정리합니다.
