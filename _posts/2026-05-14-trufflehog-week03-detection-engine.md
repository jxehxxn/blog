---
layout: post
title: "TruffleHog Week 3: 탐지 엔진 내부 — detector·verifier·entropy 삼각구도"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops
---

## 학습 목표

- TruffleHog의 3대 컴포넌트(Source, Detector, Verifier)의 책임을 분리해 이해한다.
- detector의 기본 구조(`Keywords`, `Pattern`, `FromData`)를 Go 코드로 읽는다.
- entropy 기반 보조 검출 원리를 안다.
- "왜 어떤 GitHub 토큰은 안 잡히는가" 같은 미세한 질문에 답할 수 있다.

## 1. 비유로 시작하기 — 공항 검색대

TruffleHog의 파이프라인은 **공항 보안 검색대**와 같은 구조입니다.

1. **Source** = 컨베이어 벨트 (Git 커밋, 파일, S3 객체 등을 흘려보냄)
2. **Detector** = X-ray 스캐너 (의심 패턴이 보이면 알람)
3. **Verifier** = 수동 검사관 (실제로 위험한지 직접 확인)

알람 자체는 빠르고 cheap, 수동 검사(verifier)는 비싸지만 정확합니다. 빅테크에서는 이 분리를 잘 운영하는 것이 핵심입니다.

## 2. Detector 구조 — 실제 Go 코드 보기

`trufflehog/pkg/detectors/aws/aws.go` 핵심 추상 구조:

```go
type Scanner struct{
    // 매칭 직전에 빠르게 거를 키워드들
    // 본문에 "AKIA" 같은 글자가 없으면 정규식 자체를 건너뜀
}

func (s Scanner) Keywords() []string {
    return []string{"AKIA", "ASIA"}
}

var (
    // capture group 1: 키 ID, group 2: secret access key
    keyPat = regexp.MustCompile(
        `\b((?:AKIA|ASIA)[0-9A-Z]{16})\b`,
    )
    secretPat = regexp.MustCompile(
        `\b([A-Za-z0-9+/]{40})[^A-Za-z0-9+/]`,
    )
)

func (s Scanner) FromData(
    ctx context.Context,
    verify bool,
    data []byte,
) (results []detectors.Result, err error) {
    keyMatches := keyPat.FindAllStringSubmatch(string(data), -1)
    secretMatches := secretPat.FindAllStringSubmatch(string(data), -1)

    for _, k := range keyMatches {
        for _, sec := range secretMatches {
            r := detectors.Result{
                DetectorType: detectorspb.DetectorType_AWS,
                Raw:          []byte(k[1]),
                RawV2:        []byte(k[1] + ":" + sec[1]),
            }

            if verify {
                // STS GetCallerIdentity 호출
                if isLive(k[1], sec[1]) {
                    r.Verified = true
                }
            }
            results = append(results, r)
        }
    }
    return
}
```

여기서 두 가지 중요한 관찰:

1. **`Keywords()` 사전 필터**: 정규식은 비싼 연산이라, 본문에 키워드가 없으면 정규식 자체를 건너뜁니다. 700개 디텍터를 매 파일마다 돌리려면 이 최적화가 필수.
2. **AWS는 짝(key + secret)을 함께 매칭**: secret access key 단독은 base64 같은 일반 문자열이라 FP가 폭증합니다. **반드시 페어**로 매칭하는 것이 AWS detector의 트릭.

## 3. Verifier — 그저 "맞는 것 같다"에서 "확실히 살아있다"로

같은 파일의 verifier:

```go
func isLive(keyID, secret string) bool {
    cfg := aws.NewConfig().
        WithCredentials(credentials.NewStaticCredentials(keyID, secret, "")).
        WithRegion("us-east-1")
    sess := session.New(cfg)
    svc := sts.New(sess)

    out, err := svc.GetCallerIdentity(&sts.GetCallerIdentityInput{})
    if err != nil {
        return false
    }
    // out.Account, out.Arn 등이 ExtraData로 채워짐
    return true
}
```

빅테크 관점에서 **verifier 호출은 양날의 검**입니다.

- 장점: false positive 박멸. 실제 운영에서 verified만 SOC에 보내면 큐 폭발이 없습니다.
- 단점: 호출 자체가 외부 트래픽이고, **잘못 만들면 rate limit에 걸리거나 DoS가 됨**. 자체 API에 verifier를 만든다면 7주차에서 다룰 **idempotency + caching + circuit breaker** 패턴이 필수.

자세한 verifier 설계는 [보충 2: Verifier 설계 — HTTP 호출, idempotency, rate limit, 캐싱] 포스트 참조.

## 4. Entropy 기반 보조 검출

정규식으로 잡기 어려운 시크릿이 있습니다.

- 비대칭 키의 base64 인코딩 본문
- 사내 토큰처럼 일정 prefix가 없는 랜덤 문자열

이때 **Shannon Entropy**가 유용합니다. "정상적인 영어 텍스트"는 엔트로피가 ~4.0 bit/char, "랜덤 문자열"은 ~5.7 bit/char에 가깝습니다. 임계치를 두고 거르면 후보가 됩니다.

```python
import math
def shannon_entropy(s, charset="0123456789abcdefABCDEF"):
    if not s:
        return 0
    freq = {c: s.count(c) for c in set(s)}
    n = len(s)
    return -sum((f/n) * math.log2(f/n) for f in freq.values())

# 예
print(shannon_entropy("hello world"))      # ~2.85
print(shannon_entropy("9f8a3c7b4e2d1f5a")) # ~3.5
print(shannon_entropy("aGVsbG8gd29ybGQ=")) # ~4.0 — base64
```

TruffleHog는 일부 디텍터에서 보조적으로 entropy를 사용하지만, **메인 전략은 keyword + regex + verifier**입니다. detect-secrets는 반대로 entropy 우선이라 FP가 훨씬 높습니다. 정량 비교는 [보충 1] 참조.

## 5. 왜 어떤 토큰은 안 잡히나 — 사례 분석

**상황:** 사내 token 형식이 `mycorp_<32hex>`인데 TruffleHog가 못 잡습니다.

원인 가능성:

1. **빌트인 디텍터에 mycorp가 없음** → 7주차에서 커스텀 디텍터를 작성.
2. **키워드 사전 필터 미스**: 코드에서 토큰이 base64로 인코딩됨 → DecoderName: BASE64 디텍터를 거치는지 확인.
3. **regex가 부족**: 디텍터는 있지만 `_` 뒤 길이가 변동적이면 못 잡음 → regex 수정 PR.
4. **verifier가 없어 unverified로 떨어지고 `--only-verified` 필터에 걸림** → 커스텀 verifier 추가 필요 (7주차).

## 6. 실습 과제 (실습형)

1. `trufflehog/pkg/detectors/aws/aws.go` 소스를 GitHub에서 직접 열어 읽고, `Keywords()`/`FromData()`/`verify()` 함수의 라인을 표시.
2. `pkg/detectors/github/github.go`를 같은 방식으로 분석해 AWS와 다른 점 3가지를 작성.
3. 위 Python 엔트로피 함수로 다음 문자열의 엔트로피를 출력:
   - "password123"
   - "AKIAYVP4CIPPERUVIFXG"
   - "ghp_ABC123..." (자신이 무작위로 채워서)
4. 결과를 표로 정리하고, 어떤 임계값을 두면 정상 변수명과 토큰을 구분할 수 있을지 추정.

## 7. 자가평가 퀴즈

### Q1. `Keywords()` 사전 필터의 목적은?
1. 정규식 매칭의 정확도를 높이기 위함.
2. 정규식 매칭의 비용을 줄이기 위함.
3. False positive를 자동으로 분류.
4. Verifier 우선순위 결정.

**정답:** 2번. 700 개 디텍터를 모든 파일에 매번 정규식 돌리면 GB 단위 monorepo 스캔이 며칠 걸립니다. 키워드 prefilter는 99% 이상의 케이스에서 정규식을 건너뜁니다.

### Q2. AWS detector가 key/secret을 **함께** 매칭하는 이유는?
1. AWS API가 두 값 모두 필요해서.
2. Secret access key 단독은 일반 base64 문자열이라 FP가 폭증해서.
3. 1과 2 모두.
4. 둘 다 틀림.

**정답:** 3번. 검증 시 둘이 다 필요하기도 하고, secret만 매칭하면 정상 문자열을 너무 많이 잡습니다.

### Q3. Shannon Entropy가 4.0 bit/char 이상이면 무조건 시크릿인가?
1. 그렇다.
2. 일반 영어 텍스트도 4.0을 넘을 수 있으므로 단정 불가.
3. base64 인코딩된 데이터만 4.0을 넘는다.
4. 항상 정확한 분류기.

**정답:** 2번. 엔트로피는 신호일 뿐 단독 결정 기준이 아닙니다. 그래서 보조 도구로 쓰는 것.

### Q4. 사내 토큰 `mycorp_<32hex>`를 잡고 싶을 때 첫 단계는?
1. PR 보내서 트러플 시큐리티가 추가해 줄 때까지 기다린다.
2. 커스텀 디텍터(regex + 가능하면 verifier)를 작성.
3. 모든 hex 패턴을 디텍터로 등록.
4. 사용을 중단한다.

**정답:** 2번. 7주차 주제. 1은 시간 낭비, 3은 FP 폭발, 4는 비현실적.

### Q5. Verifier 호출의 잠재적 위험은?
1. 외부 트래픽 의존, rate limit, 자체 API라면 DoS 위험.
2. 영원히 안전.
3. AWS 비용이 증가한다 (GetCallerIdentity는 무료).
4. CloudTrail 흔적이 남아 감사하기 어렵다.

**정답:** 1번. GetCallerIdentity는 무료지만 자체 API로 verifier를 만들 때는 8주차/보충 2의 패턴이 필요합니다. CloudTrail 흔적은 오히려 좋습니다 — 11주차 감사 증빙.

## 8. 다음 주차

[Week 4: Git 히스토리 깊게 파기]에서는 "이미 푸시된 시크릿"이 왜 `git rm`으로 안 지워지는지, pack file·dangling commit·BFG repo cleaner의 관계, 그리고 fork/mirror까지 추적하는 방법을 다룹니다.
