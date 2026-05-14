---
layout: post
title: "TruffleHog Week 6: 조직 규모 스캔 — GitHub Org, GitLab Group, Monorepo"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops
---

## 학습 목표

- 1,000+ 리포 환경에서 TruffleHog를 안전하게 분산 운영한다.
- GitHub Org 토큰 권한을 "최소 권한 원칙"으로 구성한다.
- Monorepo(예: Google·Meta급)의 스캔 전략을 안다.
- 결과를 중앙 저장소에 모아 메트릭으로 가공하는 파이프라인을 설계한다.

## 1. 빅테크의 리포 분포

빅테크에서 "리포가 많다"는 보통 다음 같은 분포입니다.

- 핵심 monorepo 1~3개 (수십만 파일, 수십 GB)
- 서비스별 polyrepo 수백~수천 개
- 사내 fork·archive 수백 개
- 개인 sandbox 수만 개

각각의 특성이 다르고, **모두 같은 방식으로 스캔하면 망합니다**.

## 2. GitHub Org 스캔 — 최소 권한 토큰

```bash
trufflehog github \
  --org=mycorp \
  --token=$GITHUB_PAT \
  --include-forks \
  --include-paths='.*' \
  --concurrency=32 \
  --only-verified \
  --json > scan.json
```

토큰 권한 설계 (Fine-grained PAT):
- `Contents: Read`
- `Metadata: Read`
- `Pull requests: Read` (선택)

**절대 admin 권한을 주지 마세요.** verifier가 자체 토큰을 잘못 검출해서 verified로 처리할 위험이 있고, 토큰 자체가 누출되면 org 전체가 위험합니다.

## 3. Scale-out — 어떻게 1,000 리포를 빠르게 도나

단일 머신 한 번 실행은 한계가 있습니다. 빅테크 패턴:

1. **샤딩**: 리포 목록을 N등분 후 N개 워커가 병렬 스캔.
2. **메시지 큐**: SQS/PubSub에 리포 URL을 enqueue, 워커가 dequeue.
3. **결과 집계**: 각 워커가 JSON을 S3/GCS에 적재 후, BigQuery/Athena로 통합 조회.

샤딩 예시 (간단판):

```bash
gh repo list mycorp --limit 5000 --json name --jq '.[].name' > repos.txt
split -n l/16 repos.txt shard_

for shard in shard_*; do
  (while read repo; do
    trufflehog git "https://github.com/mycorp/$repo" \
      --only-verified --json >> "output_${shard}.json"
  done < "$shard") &
done
wait
```

## 4. Monorepo 전략 — "전체 스캔은 불가능"

Google 모노레포는 수 TB. 매 PR마다 풀 스캔은 불가능합니다.

- **PR 모드**: 변경된 파일만 스캔 (5주차).
- **incremental nightly**: 어제 이후 변경분만 스캔, 결과 누적.
- **shard별 weekly 풀스캔**: 디렉토리를 50개 샤드로 나눠 매주 다른 샤드를 풀스캔.
- **trigger-driven**: secret manager API가 회전 요청을 받으면 해당 path만 스캔.

## 5. 결과 집계 — 데이터 파이프라인

```
[ Workers ] --JSON--> [ S3 raw bucket ]
                          ↓
                      [ Glue ETL → Parquet ]
                          ↓
                      [ Athena/BigQuery ]
                          ↓
                      [ Looker/Grafana ]
                          ↓
                      [ Alert: Splunk/PagerDuty ]
```

핵심 메트릭(11주차에서 깊이):
- verified count by detector
- mean time to revoke (MTTR-revoke)
- 누출 노출 시간 (commit timestamp ~ revoke timestamp)
- detector별 FP 비율 (9주차)

## 6. 빅테크 실전 사례 — Slack의 "secret scanner farm"

Slack은 발표 자료에서 사내에 TruffleHog/Gitleaks 등을 묶은 "secret scanner farm"을 운영한다고 언급한 바 있습니다.

- 매일 organization 전체 스캔
- 결과는 사내 SOAR로 전송
- 토큰 종류별로 자동 회전 플레이북 트리거

이 패턴이 빅테크의 표준입니다. 8주차 도구 연동에서 직접 만들어 봅니다.

## 7. GitLab Group / Bitbucket Workspace

```bash
# GitLab Group
trufflehog gitlab \
  --token=$GITLAB_PAT \
  --group=mycorp \
  --only-verified

# Bitbucket은 직접 지원이 약하므로 git URL 리스트로 처리
trufflehog git https://bitbucket.org/mycorp/repo --only-verified
```

## 8. 실습 과제

1. 본인 GitHub 계정에 가짜 3개 리포를 만들고, 각각 다른 형태의 가짜 토큰을 심으세요.
2. `trufflehog github --org=<your-user> --only-verified` 로 한 번에 스캔.
3. 결과 JSON을 jq로 detector별 count로 집계.
4. 만약 리포가 1,000개라면 어떻게 분산할 것인지 아키텍처 다이어그램(글로)으로 묘사.

## 9. 자가평가 퀴즈

### Q1. Org 스캔용 토큰은 어떤 권한이 적절한가?
1. admin
2. Contents: Read + Metadata: Read 정도
3. owner
4. 권한 없음

**정답:** 2번.

### Q2. Monorepo에서 매 PR마다 풀 스캔이 안 되는 이유는?
1. 너무 느리고 비용이 많이 든다.
2. 도구가 지원하지 않는다.
3. 정책상 금지.
4. fetch-depth가 0이면 못 한다.

**정답:** 1번.

### Q3. 메시지 큐 기반 샤딩의 장점은?
1. 워커 추가만으로 수평 확장.
2. 실패 시 재시도가 쉽다.
3. 결과 적재가 비동기.
4. 모두 맞음.

**정답:** 4번.

### Q4. 노출 시간 메트릭(commit ~ revoke)의 의미는?
1. 공격자가 토큰을 쓸 수 있던 윈도우.
2. 코드 리뷰 속도.
3. 도구 성능.
4. CI 빌드 시간.

**정답:** 1번. 비즈니스 임팩트 평가의 핵심 지표.

### Q5. shard별 weekly 풀스캔의 효과는?
1. 매주 monorepo 전체를 한 번씩 도는 효과 + 부담 분산.
2. 의미 없음.
3. 매주 같은 부분만 본다.
4. 디스크 용량 절감.

**정답:** 1번.

## 10. 다음 주차

[Week 7: 커스텀 디텍터 작성]에서는 사내 토큰 포맷에 맞춘 detector + verifier를 Go로 직접 작성하는 워크숍입니다.
