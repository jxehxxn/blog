---
layout: post
title: "ArgoCD 심화 최종 자가평가 — 30문항"
date: 2026-05-18 22:00:00 +0900
categories: argocd advanced assessment
---

## Q1. 심화 5 영역?
1. **CMP/Generator/Hydrator/Notif/API** 2. UI 3. 무관 4. CPU

**정답: 1.**

## Q2. Source Hydrator?
1. **hydrated manifest 별도 brunch** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q3. Image Updater와 차이?
1. **Hydrator는 manifest 전체, Updater는 tag만** 2. 같음 3. UI 4. 무관

**정답: 1.**

## Q4. CMP 가치?
1. **임의 manifest 도구 통합** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q5. CMP sidecar 패턴?
1. **v2.4+ 표준, in-process deprecated** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q6. CMP discover?
1. **자동 매칭 (어느 path가 이 plugin)** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q7. Plugin generator?
1. **HTTP service로 사내 데이터 → Application** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q8. ApplicationSet 응답?
1. **JSON output.parameters 배열** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q9. Notifications 구조?
1. **service+template+trigger+subscription** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q10. Tier별 알람?
1. **Tier별 escalation** 2. 같음 3. UI 4. 무관

**정답: 1.**

## Q11. Hook 4 위치?
1. **PreSync/Sync/PostSync/SyncFail** 2. UI 3. 무관 4. 1개

**정답: 1.**

## Q12. BeforeHookCreation?
1. **다음 sync 때 이전 hook 삭제** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q13. PreSync 대표?
1. **DB migration** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q14. Dex 역할?
1. **SSO bridge (OAuth/SAML→OIDC)** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q15. group claim 가치?
1. **RBAC → IDP에 사용자 관리 위임** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q16. CI 용 권장?
1. **Project JWT 짧은 만료** 2. SSO 3. UI 4. 무관

**정답: 1.**

## Q17. ArgoCD Operator?
1. **다중 ArgoCD instance** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q18. Multi-tenant 강점?
1. **isolation + blast radius** 2. 단순 3. UI 4. 무관

**정답: 1.**

## Q19. Master+Team ArgoCD?
1. **Master가 Team CR 관리** 2. 무관 3. UI 4. 빠름

**정답: 1.**

## Q20. Matrix+Plugin?
1. **CMDB + git 결합** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q21. Multi-source App?
1. **chart vendor + values 사내** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q22. PreservedFields?
1. **재생성 시 일부 field 보존** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q23. API 3종?
1. **gRPC+REST+CLI** 2. UI 3. 무관 4. 1개

**정답: 1.**

## Q24. Backstage 통합?
1. **service catalog deploy 버튼** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q25. Token 분리?
1. **사람/CI/automation 최소 권한** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q26. Webhook 가치?
1. **git push 즉시 sync** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q27. Rate limit?
1. **사내 gateway 보강** 2. 내장 3. UI 4. 무관

**정답: 1.**

## Q28. Flux→ArgoCD Kustomization?
1. **Application path 기반** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## Q29. Migration timeline?
1. **6개월 정도** 2. 1주 3. 1년+ 4. 무관

**정답: 1.**

## Q30. 자동화 도구?
1. **일부만, 수동 검증 필수** 2. 모두 자동 3. UI 4. 무관

**정답: 1.**

다음: Argo Events.
