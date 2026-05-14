---
layout: post
title: "TruffleHog 보충 3: Pre-receive hook으로 monorepo에 차단막 설치하기"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops supplement git
---

5주차 4계층 방어선에서 짧게 언급한 **server-side pre-receive hook**을 깊게 다룹니다.

## 1. 왜 server-side hook이 필요한가

클라이언트 사이드 pre-commit은 `--no-verify`로 쉽게 우회됩니다. CI도 PR로 들어와야 작동합니다. 그런데 사내 monorepo에 누군가 **직접 push** 한다면? pre-receive hook은 **서버에서 거부**할 마지막 방어선입니다.

## 2. 작동 위치

- GitHub Enterprise Server, GitLab self-hosted, Bitbucket Server에서 지원.
- GitHub.com (SaaS)은 직접 hook을 제공하지 않음 → Branch protection + ruleset으로 대체.

## 3. 기본 구조

`hooks/pre-receive` 스크립트:

```bash
#!/usr/bin/env bash
set -euo pipefail

zero='0000000000000000000000000000000000000000'

while read old_sha new_sha ref; do
  if [[ "$new_sha" == "$zero" ]]; then
    continue  # branch deletion
  fi

  if [[ "$old_sha" == "$zero" ]]; then
    range="$new_sha"        # new branch: scan all
  else
    range="$old_sha..$new_sha"
  fi

  # 임시로 차분만 추출
  changes=$(git rev-list "$range")

  echo "Scanning $range..."
  if ! trufflehog git file://. \
        --since-commit "$old_sha" \
        --branch "$ref" \
        --only-verified \
        --fail \
        --no-update; then
    echo "❌ Verified secret detected. Push rejected."
    exit 1
  fi
done
```

핵심:
- 모든 ref·shar 쌍을 순회.
- `--only-verified --fail`로 verified면 종료 코드 1 → push reject.
- `--no-update`로 외부 의존성 제거 (필수).

## 4. 성능 — monorepo에서 살아남기

기본 구현은 매 push마다 전체 히스토리를 보기 쉬워 느려집니다.

최적화:
1. **delta only**: `--since-commit=$old_sha`만 스캔.
2. **timeout**: 30초 내 끝나지 않으면 fail-open(우회 가능) 또는 fail-closed(거부) 정책 결정.
3. **병렬성**: `--concurrency=8`.
4. **detector 한정**: high-confidence 디텍터(AWS, GitHub, Stripe)만 켜고 나머지는 비동기 잡으로.

## 5. 운영 함정 5선

1. **hook 실패 시 push 차단의 책임**: 도구가 죽으면 개발 전체가 멈춤 → 모니터링과 watchdog 필수.
2. **권한 분리**: hook 내부에서 외부 API 호출 (verifier) → 인터넷 접근 권한 필요. 보안 검토 필수.
3. **로그 보존**: hook 실행 로그를 별도 적재. push 거부 사유 분석에 사용.
4. **bypass 절차**: emergency push 절차를 사전에 정의. "도구 마비 시 어떻게?"
5. **테스트**: hook 변경마다 staging server에서 부하 테스트.

## 6. GitHub.com 대안 — Ruleset + Required CI

GitHub.com에는 pre-receive hook이 없지만 다음으로 대체:

- **Branch protection rules**: 모든 PR 머지 전 CI 통과 필수.
- **GitHub Ruleset**: 조직 단위로 정책 강제.
- **Push protection**: GitHub Secret Scanning이 push 시점에 일부 known token을 차단 (별도 기능).

CI 기반 차단은 머지 시점에만 작동하므로, 직접 push가 차단되지 않습니다. 그래서 **branch protection을 통한 PR 강제**가 필수.

## 7. 실습 과제

1. GitLab self-hosted (Docker)로 테스트 환경 구축.
2. 위 pre-receive hook 배포.
3. 가짜 verified 토큰 push 시 reject 동작 확인.
4. timeout 30초 정책으로 큰 모노레포 시뮬레이션 (1만 커밋 합성) → 성능 측정.
5. 결과 보고서 작성: 평균 hook 실행 시간 + 거부 정확도.

## 8. 결론

Pre-receive hook은 가장 깊은 방어선이지만 운영 비용이 큽니다. 빅테크 환경에선 보통 다음 우선순위로 도입합니다.

1. pre-commit (cheap)
2. PR CI (강제)
3. Org scan (사후)
4. Pre-receive (보충, 운영 여력 있는 경우)
