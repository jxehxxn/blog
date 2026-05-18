---
layout: post
title: "IaC 최종 자가평가 — 30문항 종합 퀴즈 (정답+해설)"
date: 2026-05-18 16:00:00 +0900
categories: iac terraform crossplane platform senior-series assessment
---

12주 학습자의 종합 평가. 4지선다 30문항.

## 섹션 A — Terraform 기초 (Q1~Q10)

### Q1. IaC의 4대 가치에 속하지 않는 것?
1. Declarative
2. Idempotent
3. Versioned
4. **Mutable**

**정답: 4.**

### Q2. Terraform lifecycle 4단계?
1. **init→plan→apply→destroy**
2. plan→init→apply
3. apply→plan
4. 무관

**정답: 1.**

### Q3. count vs for_each 안전?
1. count
2. **for_each (키 기반)**
3. 같음
4. 무관

**정답: 2.**

### Q4. Data source 의 용도?
1. **기존 자원 조회**
2. 새 자원 생성
3. backup
4. 무관

**정답: 1.**

### Q5. State 파일 위험?
1. **평문 비밀 + 동시 수정 충돌**
2. 디스크
3. UI
4. 무관

**정답: 1.**

### Q6. Remote backend 필수 컴포넌트?
1. **저장소 + lock**
2. UI
3. CDN
4. 무관

**정답: 1.**

### Q7. workspaces vs 디렉토리 분리 빅테크 표준?
1. workspaces
2. **디렉토리 분리**
3. 같음
4. 무관

**정답: 2.**

### Q8. import 용도?
1. **기존 자원을 state에 등록**
2. 새 자원
3. backup
4. 무관

**정답: 1.**

### Q9. Module 디자인 5계명 가장 위험 위반?
1. **version 미고정**
2. README 없음
3. 변수명
4. 무관

**정답: 1.**

### Q10. Terragrunt 가치?
1. **backend/provider/dependency 중복 제거**
2. UI
3. 빠른 CLI
4. 무관

**정답: 1.**

## 섹션 B — 워크플로 / Crossplane (Q11~Q20)

### Q11. 로컬 apply 가장 큰 위험?
1. **비인가 변경 + audit 부재**
2. 빌드
3. UI
4. 무관

**정답: 1.**

### Q12. Atlantis 가치?
1. **PR 기반 plan/apply + 권한 집중**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q13. OIDC + IAM role assume 가치?
1. **정적 키 제거**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q14. Crossplane 핵심 아이디어?
1. **K8s를 universal control plane**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q15. MR이란?
1. **클라우드 자원의 K8s CRD 인스턴스**
2. Pod
3. Helm
4. 무관

**정답: 1.**

### Q16. Continuous reconcile 효과?
1. **콘솔 변경 자동 복구**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q17. Composition 가치?
1. **여러 MR을 묶어 사내 패키지화**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q18. v1.14+ Composition Functions?
1. **외부 컨테이너로 임의 로직**
2. UI
3. 무료
4. 무관

**정답: 1.**

### Q19. XRD vs Claim?
1. XRD cluster-scoped, **Claim namespace-scoped 사용자 인터페이스**
2. 같은 것
3. 반대
4. 무관

**정답: 1.**

### Q20. Crossplane State 위치?
1. S3
2. **K8s etcd**
3. Git
4. 무관

**정답: 2.**

## 섹션 C — 운영 / 거버넌스 (Q21~Q30)

### Q21. Provider 버전 고정 가치?
1. **예기치 못한 destroy 차단**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q22. Drift 가장 흔한 원인?
1. **콘솔 직접 변경**
2. cosmic ray
3. UI
4. 무관

**정답: 1.**

### Q23. Terraform drift 감지?
1. apply
2. **plan -detailed-exitcode (exit 2)**
3. destroy
4. 무관

**정답: 2.**

### Q24. 긴급 변경 시 Crossplane 권장?
1. **reconcile 일시 정지 (paused annotation)**
2. 그냥 콘솔
3. 무관
4. 무시

**정답: 1.**

### Q25. OPA Policy as Code 가치?
1. **사전 차단**
2. UI
3. 빠름
4. 무관

**정답: 1.**

### Q26. Infracost 가치?
1. **PR 시점 비용 가시화**
2. 빠름
3. UI
4. 무관

**정답: 1.**

### Q27. default_tags 가치?
1. **모든 자원 자동 tag → cost allocation 정확**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q28. SOC 2 CC8.1 매핑 핵심?
1. **PR=Change Request, Git history=evidence**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q29. Audit evidence 보존 권장?
1. **7년 (가장 긴 규제 기준)**
2. 1일
3. 1주
4. 무관

**정답: 1.**

### Q30. Terraform/Crossplane 하이브리드의 핵심 규율?
1. **자원 owner 명확 (둘이 같은 자원 만지면 안 됨)**
2. 무관
3. 둘 다 만지기
4. UI

**정답: 1.**

## 채점

- 28~30: Senior Platform 수준.
- 24~27: 미드, governance/operations 보강.
- 18~23: 주니어, Crossplane/모듈 추가.
- ~17: 핵심 챕터 재복습.

수고하셨습니다. 다음 코스는 Argo Workflows / Tekton.
