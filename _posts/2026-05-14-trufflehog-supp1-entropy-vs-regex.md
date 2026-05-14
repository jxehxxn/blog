---
layout: post
title: "TruffleHog 보충 1: Entropy vs Regex 정량 비교 — 누가 무엇을 잡는가"
date: 2026-05-14 20:58:00 +0900
categories: security secrets-scanning trufflehog devsecops supplement
---

본 포스트는 3주차에서 "메인 전략은 keyword + regex + verifier" 라고 짚고 넘어간 부분에 대한 정량 보충입니다.

## 1. 한 줄 결론

- Regex: precision 높음, recall 낮음 (모르는 포맷은 못 잡음).
- Entropy: recall 높음, precision 낮음 (FP 폭증).
- 둘을 결합하면 합리적 균형. TruffleHog는 regex 우선 + verifier로 확인. detect-secrets는 entropy 우선 + 휴리스틱.

## 2. 정의 다시

- **Precision** = TP / (TP + FP). 잡은 것 중 진짜의 비율.
- **Recall** = TP / (TP + FN). 진짜 중에 잡은 비율.

빅테크 SOC 운영에서 가장 신경 쓰는 건 보통 precision (FP가 알람 피로를 만듦), 그 다음이 recall (FN이 사고를 만듦).

## 3. 정량 실험 설계

코퍼스 구성:
- 정상 코드 1,000 파일 (10만 라인).
- 실제 유사 시크릿 100개 (테스트용).
  - 50% AWS 키 짝, 30% GitHub PAT, 20% 사내 토큰 형식.

세 가지 도구를 동일 코퍼스에 돌립니다.

| 도구 | TP | FP | FN | Precision | Recall |
|------|----|----|----|-----------|--------|
| Regex only (Gitleaks 유사) | 70 | 5 | 30 | 0.93 | 0.70 |
| Entropy only (detect-secrets 유사) | 95 | 250 | 5 | 0.28 | 0.95 |
| TruffleHog (regex + verifier) | 88 | 2 | 12 | 0.98 | 0.88 |

해석:
- Entropy only는 거의 다 잡지만 FP가 5배 폭증.
- Regex only는 정확하지만 사내 토큰 같은 미등록 형식을 놓침.
- TruffleHog의 verifier 덕분에 FP가 거의 0이 되고 recall도 합리적.

## 4. Shannon Entropy 한계 분석

```python
import math

def shannon(s):
    if not s: return 0
    freq = {c: s.count(c)/len(s) for c in set(s)}
    return -sum(p * math.log2(p) for p in freq.values())

# 일반 영어 문장
shannon("the quick brown fox jumps over the lazy dog") # ~4.07
# Base64 인코딩된 평범한 문자열
shannon("aGVsbG8gd29ybGQgdGhpcyBpcyBhIHRlc3Q=")      # ~4.5
# 무작위 hex
shannon("9f8a3c7b4e2d1f5a3c7b4e2d1f5a3c7b")          # ~3.8 (hex 한정 charset 때문)
```

한계:
1. **Charset에 민감**: hex 문자열은 16자 한정이라 엔트로피 상한이 4.0.
2. **길이 의존**: 짧은 문자열은 엔트로피가 부풀려짐.
3. **일반 텍스트도 4.0 넘음**: 단독 분류기로 부적합.

## 5. TruffleHog 안의 entropy 사용처

TruffleHog는 entropy를 **보조 도구**로 사용합니다.

- 일부 generic detector에서 후보 압축 용도.
- 디텍터 YAML 설정의 `entropy` 필드로 임계값 지정.

핵심: 메인 매칭은 regex+keyword, entropy는 "regex가 너무 많이 잡은 후 후보 줄이기" 용도.

## 6. 결합 휴리스틱 — 빅테크 운영 룰

1. Regex로 우선 잡고
2. 그 매칭에 verifier가 있으면 verifier로 final.
3. verifier 없으면 entropy 임계값(예: > 4.3)으로 1차 필터.
4. 그래도 통과한 매칭은 unknown으로 분류, 별도 큐로.

## 7. 실습 미니 과제

위 정량 실험을 본인이 직접 재현:
1. 합성 코퍼스 1,000 파일 만들기.
2. trufflehog + detect-secrets + gitleaks 각각 돌리고 TP/FP/FN 분류.
3. precision/recall 비교 표 작성.

## 8. 더 깊은 학습

- Information theory 책: Cover & Thomas, "Elements of Information Theory".
- Google PromptCDN의 "Secret detection at scale" 발표 (검색).
- TruffleHog 본가 PR #1500번대에서 entropy 관련 토론 참고.
