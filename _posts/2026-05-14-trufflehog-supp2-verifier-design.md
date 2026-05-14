---
layout: post
title: "TruffleHog 보충 2: Verifier 설계 — HTTP, idempotency, rate limit, 캐싱"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops supplement
---

3주차에서 "양날의 검"이라 부른 verifier를 빅테크 운영 수준에서 설계하는 법을 정리합니다.

## 1. Verifier가 만드는 7가지 위험

1. 외부 API에 의존 — 장애 전파.
2. 같은 토큰 N회 검증 — 호출 폭증.
3. 사내 API 호출 시 rate limit 침범.
4. CloudTrail 등 감사 로그가 verifier 호출로 노이즈화.
5. 검증 호출 자체가 토큰을 "탈취 가능 트래픽"으로 노출.
6. 잘못 설계하면 verifier가 자기 자신을 trigger.
7. Auth 정보를 잘못 로깅해 시크릿 누출 표면을 확대.

각각의 대응을 설계 패턴으로 정리합니다.

## 2. 패턴 1: 캐싱 (Memoization)

같은 (token hash) → 같은 결과. 5분 TTL Redis 또는 in-memory.

```go
type VerifyCache interface {
    Get(key string) (result *VerifyResult, ok bool)
    Set(key string, result *VerifyResult, ttl time.Duration)
}

func tokenKey(token string) string {
    h := sha256.Sum256([]byte(token))
    return hex.EncodeToString(h[:])
}
```

주의: 토큰 원본을 키로 쓰면 메모리·로그에 평문이 남음. **반드시 해시**.

## 3. 패턴 2: Rate limiter

```go
import "golang.org/x/time/rate"

var lim = rate.NewLimiter(rate.Limit(50), 100) // 50 RPS, burst 100

func verify(ctx context.Context, token string) {
    if err := lim.Wait(ctx); err != nil { return }
    // HTTP call
}
```

빅테크 패턴: detector별 limiter 분리. AWS는 STS quota 충분, 사내 API는 좁게.

## 4. 패턴 3: 회로 차단 (Circuit Breaker)

`gobreaker` 또는 직접:

```go
cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "mycorp-verifier",
    MaxRequests: 3,
    Interval:    60 * time.Second,
    Timeout:     30 * time.Second,
    ReadyToTrip: func(c gobreaker.Counts) bool {
        return c.ConsecutiveFailures > 5
    },
})

_, err := cb.Execute(func() (interface{}, error) {
    return doHTTP(token)
})
```

회로가 열리면 verifier 호출이 **즉시 fail-fast**로 처리되어 도구 전체가 멈추지 않습니다.

## 5. 패턴 4: Idempotency

동일 토큰을 여러 detector·여러 파일에서 만나면 검증 결과는 같아야 합니다. caching이 이를 자연스럽게 보장하지만, **분산 환경에서는 Redis** 같은 공유 캐시가 필요합니다.

## 6. 패턴 5: 검증 trace를 별도 audit log로

검증 호출이 사내 API 정상 트래픽과 섞이면 분석이 어렵습니다.
- 별도 HTTP 헤더 `X-Verifier-Source: trufflehog` 추가.
- 사내 audit DB에 `purpose=secret-verification` 태그.

## 7. 패턴 6: minimum-permission token

verifier 자체가 쓰는 토큰은 **read-only**여야 합니다. 누군가 verifier를 인수해도 권한이 거의 없어야 안전.

## 8. 패턴 7: dry-run 모드

CI에서는 verifier가 호출 안 됨을 보장하는 mode (예: 사내 API 장애 시):

```bash
trufflehog filesystem . --no-verification
```

검출만 하고 verifier 호출 0건. 사후 별도 verifier 잡으로 처리.

## 9. 전체 verifier 라이프사이클

```
[token candidate]
   ↓
[hash & cache lookup] ── hit ──→ [return cached]
   ↓ miss
[circuit breaker check] ── open ──→ [return unknown]
   ↓ closed
[rate limiter wait]
   ↓
[http call with audit headers]
   ↓
[cache set]
   ↓
[return result]
```

## 10. 실습 미니 과제

1. 위 라이프사이클을 Go로 구현 (skeleton).
2. unit test:
   - 캐시 hit
   - rate limit 초과 시 backoff
   - circuit open 시 fail-fast
3. integration test로 100k 가짜 토큰 input → verifier 호출 횟수가 캐시로 줄어드는지 측정.

## 11. 결론

Verifier는 도구의 정확도를 결정짓는 한편, 잘못하면 도구를 운영 무기로 전락시킵니다. 위 7가지 패턴은 모두 함께 적용해야 빅테크에서 안전하게 살아남습니다.
