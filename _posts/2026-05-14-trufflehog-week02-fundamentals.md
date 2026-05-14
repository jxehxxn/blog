---
layout: post
title: "TruffleHog Week 2: 설치부터 첫 스캔까지, 그리고 출력 한 줄도 빠짐없이 읽기"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops
---

## 학습 목표

- TruffleHog v3를 3가지 방식(바이너리/Docker/소스 빌드)으로 설치한다.
- 첫 스캔을 git, filesystem, docker 대상 각각에서 실행한다.
- 출력 JSON의 모든 필드를 한 줄씩 해석한다.
- `--only-verified`, `--results`, `--include-detectors` 같은 핵심 플래그의 의미를 안다.

## 1. 설치 — 3가지 경로

### (a) 바이너리 (가장 빠름)

```bash
# Linux/macOS
curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh \
  | sh -s -- -b /usr/local/bin

# 버전 확인
trufflehog --version
# trufflehog 3.x.x
```

### (b) Docker (CI/CD 표준)

```bash
docker run --rm -it trufflesecurity/trufflehog:latest \
  github --repo https://github.com/trufflesecurity/test_keys
```

빅테크 환경에서는 **사내 컨테이너 레지스트리에 이미지를 미러링**합니다. 외부 Docker Hub rate limit에 묶이면 CI가 끊깁니다. 5주차에서 다시 다룹니다.

### (c) 소스 빌드 (커스텀 디텍터 개발용)

```bash
git clone https://github.com/trufflesecurity/trufflehog.git
cd trufflehog
go build -o trufflehog ./
```

## 2. 첫 스캔 — Hello, Secret

TruffleHog는 의도적으로 만든 테스트 리포가 있습니다.

```bash
trufflehog git https://github.com/trufflesecurity/test_keys --only-verified
```

출력은 사람이 읽는 컬러 텍스트 또는 `--json` 옵션으로 JSON. 빅테크에서는 항상 JSON으로 받아 파이프라인에 넣습니다.

```bash
trufflehog git https://github.com/trufflesecurity/test_keys --only-verified --json
```

## 3. 출력 한 줄도 빠짐없이 읽기

`--json` 출력 한 건 예시:

```json
{
  "SourceMetadata": {
    "Data": {
      "Git": {
        "commit": "fbc14303ffbf8fb1c2c1914e8dda7d0121633aca",
        "file": "keys",
        "email": "counter@counter-mbp.local",
        "repository": "https://github.com/trufflesecurity/test_keys",
        "timestamp": "2022-06-16 10:55:36 -0700 PDT",
        "line": 4
      }
    }
  },
  "SourceID": 1,
  "SourceType": 7,
  "SourceName": "trufflehog - git",
  "DetectorType": 2,
  "DetectorName": "AWS",
  "DecoderName": "PLAIN",
  "Verified": true,
  "Raw": "AKIAYVP4CIPPERUVIFXG",
  "RawV2": "",
  "Redacted": "AKIAYVP4CIPPERUVIFXG",
  "ExtraData": {
    "account": "595918472158",
    "arn": "arn:aws:iam::595918472158:user/canarytokens.com/canarytoken/v..."
  }
}
```

필드별 의미를 빠짐없이 해석합니다.

- `SourceMetadata`: 어디서 발견했는지. Git이면 commit/file/line, S3면 bucket/key, Docker면 image/layer.
- `commit`: 시크릿이 처음 들어온 커밋. 4주차에서 다룰 "히스토리 추적"의 시작점.
- `file`, `line`: 정확한 위치. 인시던트 대응 시 PR 링크로 변환됩니다.
- `email`: 커밋 작성자. 책임 추적용이지만 **비난용이 아닙니다** — 보복 문화를 피하는 것이 빅테크 보안의 기본 원칙.
- `timestamp`: 커밋 시각. 누출 윈도우(window of exposure) 계산용.
- `SourceID`, `SourceType`: TruffleHog 내부 식별자. 다중 소스 동시 스캔 시 구분.
- `DetectorType`, `DetectorName`: 어떤 디텍터가 잡았는지. AWS, GitHub, Stripe 등 700+ 종류.
- `DecoderName`: PLAIN/BASE64/UTF16 등. base64 인코딩된 비밀도 디코더가 풀어서 검사합니다.
- `Verified`: **가장 중요한 필드.** true면 실제 살아있는 키, false면 패턴은 맞지만 검증 실패(이미 폐기됐거나 verifier가 없는 디텍터).
- `Raw`: 원본 시크릿 값. **이걸 로그에 그대로 남기는 것은 보안 사고를 더 키울 수 있습니다.** 항상 `Redacted` 사용 또는 `--no-verification`+해시.
- `ExtraData`: verifier가 추가로 알아낸 정보. AWS의 경우 account ID, IAM principal까지 표시됩니다. **이게 verifier의 진짜 가치**: "어떤 키"가 아니라 "어떤 권한을 가진 누구의 키"인지 즉시 압니다.

## 4. 핵심 플래그 — 빅테크에서 매일 쓰는 것

```bash
# 1. 검증된 것만
trufflehog git <url> --only-verified

# 2. 특정 디텍터만 (성능 최적화, FP 줄이기)
trufflehog git <url> --include-detectors aws,github,stripe

# 3. 결과 카테고리 필터 (verified, unverified, unknown)
trufflehog git <url> --results=verified,unknown

# 4. 동시성
trufflehog git <url> --concurrency 16

# 5. 파일 시스템 스캔
trufflehog filesystem /home/user/code --no-update

# 6. Docker 이미지 스캔
trufflehog docker --image=alpine:3.18 --only-verified
```

빅테크 운영 팁: `--no-update`는 CI 환경에서 필수입니다. CI 시작마다 디텍터 업데이트 요청을 보내면 (a) 외부 의존성 (b) 결과의 비결정성이 생깁니다. 디텍터 업데이트는 별도 주간 잡(weekly cron)으로 관리합니다.

## 5. 첫 스캔을 의미 있게 만들기 — 실험

테스트 리포를 fork해서 가짜 시크릿을 더 넣고 어떻게 잡히는지 봅니다.

```bash
# 사용자 리포에 가짜 GitHub 토큰 패턴 추가
echo "GITHUB_TOKEN=ghp_FAKE1234567890abcdefghijklmnopqrstuv" > test.txt
git add test.txt
git commit -m "test"

trufflehog filesystem . --json | jq 'select(.DetectorName=="Github")'
```

- `ghp_` 프리픽스가 GitHub PAT 패턴과 매칭됩니다.
- `Verified` 필드는 false일 것입니다 (가짜이므로 verifier API 호출 실패).
- 이 차이를 직접 보는 것이 핵심.

## 6. 실습 과제

1. 로컬에 빈 리포를 만들고 다음 4종의 가짜 시크릿을 각각 다른 커밋에 넣으세요.
   - AWS Access Key 형태 (`AKIA...`)
   - GitHub PAT 형태 (`ghp_...`)
   - Slack Webhook 형태 (`https://hooks.slack.com/services/...`)
   - 평문 JSON 안의 `password` 필드
2. `trufflehog git file://$(pwd) --json` 결과를 캡처.
3. 각 항목의 `Verified`, `DetectorName`, `commit` 필드를 표로 정리.
4. **왜** password 평문이 잡히지 않거나 unverified로 나오는지 한 단락으로 설명.

## 7. 자가평가 퀴즈

### Q1. `--only-verified`를 빼면 어떤 일이 생기나?
1. 모든 매칭이 출력되어 FP가 늘어난다.
2. 스캔 속도가 빨라진다.
3. JSON이 더 간결해진다.
4. 변화 없음.

**정답:** 1번. unverified와 unknown까지 출력됩니다. CI에서는 일반적으로 `--only-verified`로 시작하고, 별도 주간 잡으로 unverified를 검토합니다.

### Q2. `ExtraData.account`는 어떻게 채워지나?
1. TruffleHog 내부 데이터베이스 조회
2. Verifier가 STS GetCallerIdentity를 호출해 응답에서 추출
3. 사용자가 수동 입력
4. AWS Console API를 무단으로 호출

**정답:** 2번. 합법적인 검증 호출(자신의 자격증명이 어디에 매핑되는지 묻는 STS API)을 통해 얻습니다. **이 호출이 CloudTrail에 흔적을 남깁니다** — 11주차 거버넌스에서 다룹니다.

### Q3. `Raw` 필드를 로그에 남기면 안 되는 이유는?
1. 디스크 용량을 낭비한다.
2. 시크릿 원본이 평문으로 로그에 남아 누출이 두 배로 커진다.
3. JSON이 깨진다.
4. 법적으로 금지되어 있다.

**정답:** 2번. 로그도 시크릿 누출 표면(attack surface)입니다. 항상 `Redacted` 또는 해시(예: SHA256) 형태로 남기는 것이 빅테크 표준입니다.

### Q4. CI에서 `--no-update`가 권장되는 이유는?
1. 결정적 빌드 + 외부 의존성 제거.
2. 디텍터를 일부러 옛 버전으로 고정해 호환성을 챙긴다.
3. 네트워크 트래픽을 줄여 비용 절감.
4. 1과 3 모두.

**정답:** 4번. 결정성 + 외부 의존성 차단이 본질이지만 트래픽 절감도 부수효과입니다.

### Q5. `DecoderName: BASE64`가 의미하는 것은?
1. 시크릿이 base64로 인코딩된 채로 코드에 있었고, 디코더가 풀어서 검출.
2. 시크릿 자체가 base64 알파벳으로만 구성되어 있다.
3. 출력이 base64로 인코딩되어 있다.
4. 도구가 오류를 일으켰다.

**정답:** 1번. TruffleHog는 base64/UTF16/hex 디코더를 통해 인코딩된 시크릿도 탐지합니다.

## 8. 다음 주차

[Week 3: 탐지 엔진 내부]에서는 detector–verifier–entropy 삼각구도가 어떻게 실제 코드에서 구현되는지 GitHub의 trufflehog Go 소스를 직접 따라 읽습니다. "왜 어떤 키는 잡히고 어떤 키는 못 잡히는가"의 근본 원인이 보일 것입니다. 심화는 **보충 1: Entropy vs Regex 정량 비교**, **보충 2: Verifier 설계** 포스트를 함께 읽으세요.
