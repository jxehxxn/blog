---
layout: post
title: "GoCD 최종 자가평가 — 30문항 종합 (정답+해설)"
date: 2026-05-19 09:00:00 +0900
categories: cicd gocd assessment
---

12주 학습자의 종합 평가.

## 섹션 A — 기초 (Q1~Q10)

### Q1. GoCD 모기업?
1. **ThoughtWorks** 2. Google 3. AWS 4. CNCF

**정답: 1.**

### Q2. 모태 책?
1. **Continuous Delivery (Humble & Farley)**
2. Phoenix Project 3. DevOps Handbook 4. SRE Book

**정답: 1.**

### Q3. 시그니처 기능?
1. **Value Stream Map (VSM)**
2. UI 색 3. 빠른 빌드 4. plugin

**정답: 1.**

### Q4. 4계층?
1. **Pipeline > Stage > Job > Task**
2. UI 3. 무관 4. 다른 순서

**정답: 1.**

### Q5. Stage 간 관계?
1. **순차**
2. 병렬 3. 무관 4. 임의

**정답: 1.**

### Q6. 같은 stage Job 간?
1. **병렬**
2. 순차 3. 무관 4. 무작위

**정답: 1.**

### Q7. Fetch Artifact?
1. **stage/pipeline 간 산출물 명시적 전달**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q8. Manual approval 표준?
1. **prod stage + RBAC role**
2. 모든 stage 3. UI 4. 무관

**정답: 1.**

### Q9. Material 가장 흔한?
1. **git**
2. svn 3. p4 4. 무관

**정답: 1.**

### Q10. Webhook URL?
1. **/go/api/webhooks/github/notify**
2. /webhook 3. /trigger 4. 무관

**정답: 1.**

## 섹션 B — 운영 (Q11~Q20)

### Q11. Pipeline Group?
1. **namespace + RBAC 단위**
2. UI 색 3. 빠름 4. 무관

**정답: 1.**

### Q12. Environment 책임?
1. **pipeline + agent + 변수 묶음**
2. UI 3. 무관 4. 빠름

**정답: 1.**

### Q13. RBAC 3종?
1. **admins / operate / view**
2. UI 3. 무관 4. read/write/exec

**정답: 1.**

### Q14. Pipeline as Code 가치?
1. **PR + audit + 자동 반영**
2. UI 색 3. 빠름 4. 무관

**정답: 1.**

### Q15. 표준 PaC plugin?
1. **yaml-config-plugin**
2. groovy 3. json 4. UI

**정답: 1.**

### Q16. VSM 의미?
1. **commit → prod 경로 시각화**
2. UI 색 3. 빠름 4. 무관

**정답: 1.**

### Q17. Fan-out?
1. **1 upstream → N downstream**
2. N → 1 3. UI 4. 무관

**정답: 1.**

### Q18. Fan-in?
1. **N upstream 모두 성공 → 1 downstream**
2. 1 → N 3. UI 4. 무관

**정답: 1.**

### Q19. 순환 dependency?
1. **GoCD 거부**
2. 허용 3. UI 4. 무관

**정답: 1.**

### Q20. Approval type?
1. **success / manual**
2. UI 3. 무관 4. auto/cron

**정답: 1.**

## 섹션 C — 심화 + 비교 (Q21~Q30)

### Q21. Timer cron?
1. **정기 build (nightly)**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q22. Locking?
1. **같은 pipeline 동시 1 instance**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q23. Elastic Agent 비유?
1. **일용직 (필요 시 띄움)**
2. 정직원 3. UI 4. 무관

**정답: 1.**

### Q24. Plugin 2종 (elastic)?
1. **Docker / Kubernetes**
2. AWS only 3. UI 4. 무관

**정답: 1.**

### Q25. 비용 절감 비율?
1. **70~90%**
2. 5% 3. UI 4. 무관

**정답: 1.**

### Q26. Secret 평문 위험?
1. **모든 log/UI 노출**
2. UI 만 3. 안전 4. 무관

**정답: 1.**

### Q27. Vault 통합?
1. **secret plugin + {{SECRET:...}}**
2. UI 3. file 4. 무관

**정답: 1.**

### Q28. GoCD 강점 1?
1. **VSM 내장**
2. SaaS 3. 가벼움 4. UI

**정답: 1.**

### Q29. CD 복잡 + audit?
1. **GoCD**
2. GH Actions 3. Tekton 4. 무관

**정답: 1.**

### Q30. Jenkins migration 기간?
1. **6개월 점진**
2. 1주 3. 1년+ 4. 무관

**정답: 1.**

## 채점

- 28~30: 빅테크 GoCD senior 수준.
- 24~27: 미드.
- 18~23: 주니어.
- ~17: 핵심 재복습.

12주 수고하셨습니다.
