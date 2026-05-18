---
layout: post
title: "Workflow 최종 자가평가 — 30문항 종합 (정답+해설)"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series assessment
---

12주 학습자 종합 평가.

## 섹션 A — Argo Workflows (Q1~Q10)

### Q1. Workflow entrypoint?
1. **시작 template 이름** 2. SA 3. 이미지 4. 무관

**정답: 1.**

### Q2. DAG가 steps 권장 상황?
1. **복잡한 의존성** 2. 빠름 3. UI 4. 무관

**정답: 1.**

### Q3. Output 캡처?
1. **outputs.parameters.valueFrom.path** 2. stdin 3. env 4. 무관

**정답: 1.**

### Q4. Parameter vs Artifact?
1. **Parameter 작은 텍스트, Artifact 큰 파일** 2. 같음 3. 반대 4. 무관

**정답: 1.**

### Q5. Memoize 가치?
1. **같은 입력 결과 cache** 2. UI 3. 비용 4. 무관

**정답: 1.**

### Q6. fan-out 도구?
1. **withParam/withItems/withSequence** 2. exit handler 3. mutex 4. 무관

**정답: 1.**

### Q7. Exit handler 가치?
1. **성공/실패 무관 cleanup·알람** 2. 빠름 3. UI 4. 무관

**정답: 1.**

### Q8. WorkflowTemplate 가치?
1. **재사용 가능한 사내 표준** 2. UI 3. 비용 4. 무관

**정답: 1.**

### Q9. Argo Events 3 컴포넌트?
1. **EventBus, EventSource, Sensor** 2. UI 3. 무관 4. Step/DAG/Workflow

**정답: 1.**

### Q10. Filter 가치?
1. **특정 조건 event만 trigger** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## 섹션 B — Tekton (Q11~Q20)

### Q11. Tekton 3 layer?
1. **Task / Pipeline / PipelineRun** 2. UI 3. 무관 4. Step/Workflow/Run

**정답: 1.**

### Q12. Workspace 역할?
1. **step 간 디렉토리 공유** 2. UI 3. 비용 4. 무관

**정답: 1.**

### Q13. Results 용도?
1. **작은 출력값 → 다음 Task 전달** 2. UI 3. 비용 4. 무관

**정답: 1.**

### Q14. Tekton Hub 가치?
1. **표준 Task catalog 재사용** 2. UI 3. 비용 4. 무관

**정답: 1.**

### Q15. Finally 역할?
1. **항상 실행 cleanup** 2. 빠름 3. UI 4. 무관

**정답: 1.**

### Q16. Triggers 3 컴포넌트?
1. **EventListener, TriggerBinding, TriggerTemplate** 2. UI 3. 무관 4. Source/Bus/Sensor

**정답: 1.**

### Q17. Resolver 가치?
1. **원격 Task/Pipeline 참조 (cluster 설치 없이)** 2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q18. Chains 가치?
1. **자동 서명·증명 (SLSA)** 2. UI 3. 비용 4. 무관

**정답: 1.**

### Q19. EventListener secret 검증 누락 위험?
1. **외부 누구나 trigger** 2. UI 3. 비용 4. 무관

**정답: 1.**

### Q20. enum 가치?
1. **잘못된 parameter 차단** 2. UI 3. 빠름 4. 무관

**정답: 1.**

## 섹션 C — CI/CD 통합 + 보안 (Q21~Q30)

### Q21. CI 빌드에 적합?
1. **Tekton** 2. Argo Workflows 3. 같음 4. 무관

**정답: 1.**

### Q22. ML/data 파이프라인에 적합?
1. **Argo Workflows** 2. Tekton 3. 같음 4. 무관

**정답: 1.**

### Q23. Daemonless build 도구?
1. **Kaniko/Buildah** 2. docker 3. UI 4. 무관

**정답: 1.**

### Q24. Cosign keyless 가치?
1. **정적 key 관리 부담 제거** 2. UI 3. 비용 4. 무관

**정답: 1.**

### Q25. CI/CD 책임 분리?
1. **CI는 build/manifest 변경, CD는 cluster apply** 2. CI가 모두 3. CD가 모두 4. 무관

**정답: 1.**

### Q26. Rollback 빅테크 표준?
1. **git revert** 2. CI 재실행 3. UI 4. 무관

**정답: 1.**

### Q27. SLSA Level 3 핵심?
1. **빌드 격리 + provenance 서명** 2. UI 3. 무료 4. 무관

**정답: 1.**

### Q28. PSA restricted 가치?
1. **privileged Pod 차단 → DinD 같은 위험 차단** 2. UI 3. 비용 4. 무관

**정답: 1.**

### Q29. Audit 보존 권장?
1. **7년** 2. 1일 3. 1주 4. 무관

**정답: 1.**

### Q30. Jenkins → 새 도구 마이그레이션 기간?
1. **6개월 정도 단계별** 2. 1주 3. 1년+ 4. 무관

**정답: 1.**

## 채점

- 28~30: Senior 수준.
- 24~27: 미드, CI/CD 보안 추가.
- 18~23: 주니어, Argo Events/Tekton Chains 보강.
- ~17: 핵심 재복습.

다음 코스: Observability 풀스택.
