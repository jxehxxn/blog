---
layout: post
title: "TruffleHog Week 7: 커스텀 디텍터 — 사내 토큰을 위한 detector + verifier 만들기"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops
---

## 학습 목표

- TruffleHog의 detector 인터페이스를 구현한다.
- 정규식, 키워드, decoder 구성을 실전 수준으로 결정한다.
- verifier에 HTTP 호출 + 캐싱 + 회로차단(circuit breaker)을 결합한다.
- 사내 토큰 `mycorp_<32hex>` 디텍터를 처음부터 끝까지 만든다.

## 1. 사전 — 두 가지 경로

TruffleHog는 커스텀 디텍터를 두 방법으로 받습니다.

1. **YAML 정의(no-code)**: regex만으로 충분할 때.
2. **Go 코드(plug-in 빌트인)**: verifier가 필요할 때.

YAML이 빠르지만, **빅테크 환경에서는 verifier가 거의 필수**이므로 Go 코드를 다룹니다.

## 2. YAML 정의 — 간단한 경우

```yaml
# detectors.yaml
detectors:
  - name: MyCorpToken
    keywords:
      - mycorp_
    regex:
      token: 'mycorp_[0-9a-f]{32}'
    entropy: 4.2  # 선택
```

```bash
trufflehog filesystem . --config detectors.yaml --only-verified=false
```

한계: verifier가 없어 `--only-verified` 필터에 잡히지 않습니다.

## 3. Go 코드 디텍터 — 풀스택 예제

`pkg/detectors/mycorp/mycorp.go` (사내 fork 가정):

```go
package mycorp

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "regexp"
    "strings"
    "time"

    "github.com/trufflesecurity/trufflehog/v3/pkg/common"
    "github.com/trufflesecurity/trufflehog/v3/pkg/detectors"
    "github.com/trufflesecurity/trufflehog/v3/pkg/pb/detectorspb"
)

type Scanner struct{
    client *http.Client
}

var _ detectors.Detector = (*Scanner)(nil)

var (
    keyPat = regexp.MustCompile(`\b(mycorp_[0-9a-f]{32})\b`)
)

func (s Scanner) Keywords() []string {
    return []string{"mycorp_"}
}

func (s Scanner) Type() detectorspb.DetectorType {
    return detectorspb.DetectorType_Custom // 외부 PR 시 새 enum 필요
}

func (s Scanner) Description() string {
    return "MyCorp internal API token"
}

func (s Scanner) FromData(
    ctx context.Context, verify bool, data []byte,
) (results []detectors.Result, err error) {
    matches := keyPat.FindAllStringSubmatch(string(data), -1)
    seen := map[string]struct{}{}

    for _, m := range matches {
        token := m[1]
        if _, ok := seen[token]; ok {
            continue // 중복 제거
        }
        seen[token] = struct{}{}

        r := detectors.Result{
            DetectorType: s.Type(),
            Raw:          []byte(token),
            Redacted:     token[:10] + "...",
        }

        if verify {
            ok, extra := s.verify(ctx, token)
            r.Verified = ok
            r.ExtraData = extra
        }
        results = append(results, r)
    }
    return
}

func (s Scanner) verify(ctx context.Context, token string) (bool, map[string]string) {
    client := s.client
    if client == nil {
        client = &http.Client{Timeout: 5 * time.Second}
    }

    req, _ := http.NewRequestWithContext(ctx,
        http.MethodGet,
        "https://api.mycorp.internal/v1/whoami",
        nil,
    )
    req.Header.Set("Authorization", "Bearer "+token)

    resp, err := client.Do(req)
    if err != nil {
        return false, nil
    }
    defer resp.Body.Close()
    if resp.StatusCode != 200 {
        return false, nil
    }
    body, _ := io.ReadAll(resp.Body)
    return true, map[string]string{
        "whoami": strings.TrimSpace(string(body)),
    }
}
```

## 4. Verifier — 빅테크 운영 패턴

자체 verifier는 외부 의존성을 만듭니다. 필수 보완:

### (a) 캐싱

```go
type cachedVerifier struct {
    cache *sync.Map // token -> {ok, expiresAt}
}

func (c *cachedVerifier) verify(ctx context.Context, token string) bool {
    if v, ok := c.cache.Load(token); ok {
        e := v.(entry)
        if time.Now().Before(e.expiresAt) {
            return e.ok
        }
    }
    ok := actualHTTPCall(ctx, token)
    c.cache.Store(token, entry{ok: ok, expiresAt: time.Now().Add(5 * time.Minute)})
    return ok
}
```

### (b) Rate limiting

```go
limiter := rate.NewLimiter(rate.Every(100*time.Millisecond), 5) // 50 RPS
limiter.Wait(ctx)
```

### (c) 회로 차단 (circuit breaker)

```go
// gobreaker 등 사용
cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name: "mycorp-verifier",
    Timeout: 30 * time.Second,
    ReadyToTrip: func(c gobreaker.Counts) bool {
        return c.ConsecutiveFailures > 5
    },
})
_, err := cb.Execute(func() (interface{}, error) { return doHTTP() })
```

자세한 verifier 설계는 [보충 2: Verifier 설계] 포스트.

## 5. 빌드 & 통합

```bash
# 사내 fork에서
make build
./trufflehog filesystem . --only-verified
```

PR 형태로 본가에 보낼 수도 있지만, 보통 사내 토큰은 **fork에서만 유지**합니다(내부 정보 노출 방지).

## 6. 빅테크 실전 사례 — Shopify의 detector PR 패턴

Shopify는 사내 token format을 위해 OSS TruffleHog에 PR을 보내 추가했습니다. 단, verifier는 사내 fork에만 두고 public 디텍터는 regex만 공개했습니다. 이렇게 하면:

- 외부 보안 커뮤니티가 사내 토큰 누출을 발견해 알릴 수 있고
- 사내 verifier는 내부 API URL을 외부에 노출하지 않습니다.

## 7. 실습 과제 (실습형, 길음)

1. 자체 fork에서 `pkg/detectors/mycorp/`를 만들고 위 코드를 약간 다듬어 컴파일.
2. `go test ./pkg/detectors/mycorp/...` 가능하도록 unit test 작성:
   - 유효한 토큰 → verified true
   - 만료 토큰 → verified false
   - 부분 매칭 → 매칭 안 됨
3. 5분간 1000개 토큰 검증 시 rate limiter가 잘 동작하는지 통합 테스트.
4. 회로차단이 실제로 트리거되는 시나리오 작성.
5. 위 모든 것의 결과를 한 페이지 보고서로 정리.

## 8. 자가평가 퀴즈

### Q1. YAML 정의 디텍터의 한계는?
1. 너무 빠름.
2. verifier를 붙일 수 없어 verified 필터에 안 들어옴.
3. 메모리 사용량.
4. 유지보수 어려움.

**정답:** 2번.

### Q2. Verifier에 캐싱이 필요한 이유?
1. 같은 토큰을 monorepo 곳곳에서 만나는 경우가 흔해서 호출 폭증 방지.
2. 디스크 절약.
3. 결과 정확도 향상.
4. 모두 맞음.

**정답:** 1번이 핵심. 같은 토큰을 100번 검증하지 말고 캐시.

### Q3. 사내 verifier의 회로 차단 목적은?
1. 사내 API가 일시 장애일 때 verifier가 멈춰 도구 전체를 막지 않게.
2. 토큰을 자동 회전.
3. CI 비용 절감.
4. detector 비활성화.

**정답:** 1번.

### Q4. 사내 토큰을 OSS PR로 추가할 때 best practice는?
1. regex만 공개, verifier는 사내 fork에 둠.
2. 모두 공개.
3. 절대 외부 PR 금지.
4. fork 자체를 비공개.

**정답:** 1번.

### Q5. detector의 `Keywords()`를 비워두면?
1. 모든 파일에서 정규식이 돌아 매우 느려짐.
2. 속도 향상.
3. 오류 발생.
4. 매칭 정확도 향상.

**정답:** 1번.

## 9. 다음 주차

[Week 8: 도구 연동]에서는 verifier 결과를 Splunk SIEM에 흘리고, HashiCorp Vault에 자동 회전 트리거를 걸고, Tines SOAR로 워크플로를 묶는 통합을 다룹니다.
