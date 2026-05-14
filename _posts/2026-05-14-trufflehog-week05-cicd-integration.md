---
layout: post
title: "TruffleHog Week 5: CI/CD 통합 — pre-commit, GitHub Actions, GitLab CI, Jenkins"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops
---

## 학습 목표

- 4계층 방어선(IDE → pre-commit → CI → org scan)을 설계한다.
- pre-commit hook으로 로컬에서 시크릿을 차단한다.
- GitHub Actions / GitLab CI / Jenkins 각각의 통합을 실제 YAML로 작성한다.
- "PR 차단" vs "리포팅 전용" 정책의 트레이드오프를 안다.

## 1. 빅테크의 4계층 방어선

```
[IDE 플러그인]   ← 0층: 작성 시점, 가장 cheap
   ↓
[pre-commit]     ← 1층: 커밋 시점, 로컬에서 차단
   ↓
[PR CI]          ← 2층: 머지 차단(블로킹)
   ↓
[Org scan]       ← 3층: 누락된 것 캐치, 주기적
```

빅테크 원칙:
- **최대한 왼쪽(작성 시점)에서 잡아라**. 오른쪽으로 갈수록 비용이 100배씩 증가.
- 한 층이 깨져도 다음 층이 잡도록 **defense in depth**.

## 2. Layer 1: pre-commit hook

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/trufflesecurity/trufflehog
    rev: v3.81.10  # 항상 고정 버전 사용 (재현성)
    hooks:
      - id: trufflehog
        name: TruffleHog
        description: Detect secrets in staged changes
        entry: bash -c 'trufflehog git file://. --since-commit HEAD --only-verified --fail'
        language: system
        stages: ["commit"]
```

설치:
```bash
pip install pre-commit
pre-commit install
```

이제 `git commit` 때마다 변경분이 스캔되고, verified 시크릿이 있으면 커밋이 거부됩니다.

빅테크 운영 팁:
- 사내 리포에 pre-commit 설치를 **자동화**하세요. 흔히 `bootstrap.sh` 또는 `Makefile`에 넣어 클론하자마자 활성화.
- 우회 가능성 인지: `git commit --no-verify`. 그래서 다음 층(CI)이 필요합니다.

## 3. Layer 2: GitHub Actions

`.github/workflows/trufflehog.yml`:

```yaml
name: Secret Scan
on:
  pull_request:
    branches: [main, develop]

permissions:
  contents: read
  pull-requests: write  # 결과를 PR 코멘트로 남길 때

jobs:
  TruffleHog:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 전체 히스토리 필요

      - name: Secret Scanning
        uses: trufflesecurity/trufflehog@v3.81.10
        with:
          path: ./
          base: ${{ github.event.pull_request.base.sha }}
          head: ${{ github.event.pull_request.head.sha }}
          extra_args: --only-verified --fail

      - name: Upload report on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: trufflehog-report
          path: trufflehog-output.json
```

핵심:
- `fetch-depth: 0` 없이는 히스토리를 못 봐서 스캔 범위가 망가집니다.
- `--fail`은 verified 발견 시 종료 코드 ≠ 0 → 머지 차단.
- 결과는 artifact로 업로드해 인시던트 대응 시 evidence로 보존(컴플라이언스).

## 4. Layer 2: GitLab CI

`.gitlab-ci.yml`:

```yaml
stages:
  - security

secret-scan:
  stage: security
  image:
    name: trufflesecurity/trufflehog:3.81.10
    entrypoint: [""]
  script:
    - >
      trufflehog git file://$CI_PROJECT_DIR
      --since-commit $CI_MERGE_REQUEST_DIFF_BASE_SHA
      --only-verified
      --fail
      --json > trufflehog-output.json
  artifacts:
    when: always
    paths:
      - trufflehog-output.json
    expire_in: 90 days  # 컴플라이언스 보존
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

## 5. Layer 2: Jenkins (선언형 파이프라인)

```groovy
pipeline {
  agent {
    docker {
      image 'trufflesecurity/trufflehog:3.81.10'
      args  '--entrypoint='
    }
  }
  stages {
    stage('Secret Scan') {
      steps {
        sh '''
          trufflehog git file://${WORKSPACE} \
            --since-commit ${CHANGE_TARGET} \
            --only-verified \
            --fail \
            --json > trufflehog-output.json
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'trufflehog-output.json'
        }
        failure {
          mail to: 'secops@mycorp.com',
               subject: "Secret found in ${env.JOB_NAME}",
               body: "Build ${env.BUILD_URL}"
        }
      }
    }
  }
}
```

## 6. "차단" vs "리포팅" — 정책 결정

| 정책 | 장점 | 단점 | 권장 환경 |
|------|------|------|-----------|
| 차단(block) | 머지 자체를 막아 누출 0% | 거짓 양성 시 개발 흐름 차단 | 운영 코드, 결제·인증 모듈 |
| 리포팅 | 흐름 방해 없음 | 인간이 안 보면 무의미 | 초기 도입기, 실험 리포 |
| 하이브리드 | verified=차단, unverified=리포팅 | 운영 복잡 | 대다수 빅테크가 채택 |

**빅테크 표준은 하이브리드**입니다. verified는 강제 차단, unverified/unknown은 PR 코멘트로만.

## 7. 빅테크 운영 함정 BEST 5

1. **외부 Docker Hub 의존**: rate limit으로 CI가 멈춥니다. 사내 미러 필수.
2. **fetch-depth 누락**: shallow clone이 기본인데 그러면 히스토리 비교가 안 됩니다.
3. **`--fail` 빼고 켜둠**: 결과는 나오지만 머지가 막히지 않아 실제론 미작동.
4. **verifier 호출이 사내 API rate limit을 흔듦**: 커스텀 verifier 작성 시 보충 2 참조.
5. **artifacts 보존 기간 누락**: 인시던트 대응/감사 evidence가 사라져 컴플라이언스 fail.

## 8. 실습 과제

1. 본인 GitHub 리포에 위 GitHub Actions 워크플로를 추가.
2. 가짜 AWS 키가 들어 있는 commit으로 PR 생성.
3. CI가 `Failure`로 끝나는 것을 캡처.
4. `--only-verified`를 빼고 다시 실행했을 때 차이를 비교.
5. pre-commit도 함께 활성화해 commit 시점부터 차단되는지 확인.

## 9. 자가평가 퀴즈

### Q1. `fetch-depth: 0`이 필요한 이유는?
1. 더 빠른 클론을 위해.
2. 비교용 base 커밋이 로컬에 있어야 정확한 since-commit 비교가 가능하기 때문.
3. GitHub Actions가 요구해서.
4. 의미 없는 옵션.

**정답:** 2번.

### Q2. pre-commit이 깨면 CI는 무용지물인가?
1. 그렇다.
2. 아니다. defense in depth로 CI가 잡는다.
3. 둘이 의존 관계라 같이 깨진다.
4. CI가 자동으로 pre-commit을 살린다.

**정답:** 2번.

### Q3. 하이브리드 정책에서 차단 기준은 보통?
1. 모든 매칭.
2. verified만.
3. unverified만.
4. unknown만.

**정답:** 2번. verified는 실제 살아있는 토큰이므로 거짓 양성 확률이 매우 낮음.

### Q4. `--fail` 옵션 누락의 결과는?
1. JSON이 깨진다.
2. 종료 코드가 0이라 CI는 통과되고 누출이 머지된다.
3. verifier가 꺼진다.
4. 로그 색상이 바뀐다.

**정답:** 2번.

### Q5. Docker Hub rate limit을 피하는 가장 깔끔한 방법은?
1. 익명 풀로 충분하다.
2. 사내 컨테이너 레지스트리에 미러를 두고 그것을 참조.
3. CI를 더 자주 돌린다.
4. trufflehog를 매번 새로 빌드.

**정답:** 2번.

## 10. 다음 주차

[Week 6: 조직 규모 스캔]에서는 GitHub Org 단위 1,000+ 리포에 어떻게 스캔을 분산하고, 결과를 어떻게 한곳에 모으는지를 다룹니다.
