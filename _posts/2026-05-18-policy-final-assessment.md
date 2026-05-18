---
layout: post
title: "Policy 최종 자가평가 — 30문항 종합 (정답+해설)"
date: 2026-05-18 20:00:00 +0900
categories: policy platform senior-series assessment
---

12주 학습자 종합.

## Q1. Policy as Code 핵심?
1. **일관성+GitOps+audit+scale** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q2. Admission webhook 위치?
1. **API server 검증 후** 2. etcd 후 3. UI 4. 무관

**정답: 1.**

## Q3. OPA 언어?
1. **Rego** 2. YAML 3. Python 4. JS

**정답: 1.**

## Q4. Kyverno 강점?
1. **YAML만으로 + mutation 강력** 2. Rego 3. UI 4. 무관

**정답: 1.**

## Q5. `deny[msg]` 의미?
1. **deny set에 msg 추가** 2. 함수 3. UI 4. 무관

**정답: 1.**

## Q6. Conftest 가치?
1. **Rego policy를 TF plan/manifest에 적용** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q7. ConstraintTemplate vs Constraint?
1. **Template 코드, Constraint 인스턴스** 2. 같음 3. 반대 4. 무관

**정답: 1.**

## Q8. enforcementAction 종류?
1. **deny/warn/dryrun** 2. UI 3. 무관 4. allow only

**정답: 1.**

## Q9. Audit 가치?
1. **기존 자원 위반 발견** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q10. Kyverno Mutate?
1. **요청 수정 (default 추가 등)** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q11. Generate?
1. **신규 리소스 자동 생성** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q12. VerifyImages?
1. **cosign signature 검증** 2. UI 3. 무관 4. SBOM

**정답: 1.**

## Q13. background scan?
1. **기존 자원 정기 검사** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q14. K8s 외 사용?
1. **OPA (conftest)** 2. Kyverno 3. 같음 4. 무관

**정답: 1.**

## Q15. Mutation 필수?
1. **Kyverno** 2. OPA 3. 같음 4. 무관

**정답: 1.**

## Q16. Rego 학습 부담 없음?
1. **Kyverno** 2. OPA 3. 같음 4. 무관

**정답: 1.**

## Q17. Set vs Array?
1. **Set 순서 무관, 중복 없음** 2. 같음 3. 반대 4. 무관

**정답: 1.**

## Q18. Default 가치?
1. **누락 방지** 2. 빠름 3. UI 4. 무관

**정답: 1.**

## Q19. Strategic Merge `+(name)`?
1. **없으면 추가** 2. 무시 3. UI 4. 무관

**정답: 1.**

## Q20. ForEach 가치?
1. **list 반복 mutation** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q21. mutateDigest?
1. **tag → digest immutable** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q22. SLSA attestation 검증?
1. **build provenance까지 검증** 2. signature만 3. UI 4. 무관

**정답: 1.**

## Q23. Keyless 가치?
1. **OIDC 기반 → 정적 key 부담 0** 2. UI 3. 비용 4. 무관

**정답: 1.**

## Q24. PolicyReport CR?
1. **wg-policy 표준 pass/fail** 2. UI 3. 무관 4. 임의 형식

**정답: 1.**

## Q25. Exempt 운영 규율?
1. **사유 + 만료** 2. 영구 3. UI 4. 무관

**정답: 1.**

## Q26. External Data Provider?
1. **외부 HTTP 호출** 2. K8s 내 3. UI 4. 무관

**정답: 1.**

## Q27. context cache 가치?
1. **외부 호출 부담 감소** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q28. Policy 라이프사이클?
1. **draft → dryrun → warn → enforce** 2. 한 번에 3. UI 4. 무관

**정답: 1.**

## Q29. Rollback?
1. **git revert 또는 enforcementAction dryrun** 2. 수동 3. UI 4. 무관

**정답: 1.**

## Q30. failurePolicy Fail vs Ignore?
1. **Fail 안전 (deny), Ignore 가용성** 2. 같음 3. 반대 4. 무관

**정답: 1.**

다음: Supply chain security.
