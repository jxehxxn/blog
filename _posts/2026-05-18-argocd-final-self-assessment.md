---
layout: post
title: "ArgoCD 최종 자가평가 — 30문항 종합 퀴즈 (정답+해설)"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes assessment
---

12주를 마친 학습자가 본인 수준을 가늠할 종합 퀴즈입니다. 모든 문항은 4지선다, 정답 + 해설 포함.

## 섹션 A — 기초 (Q1~Q10)

### Q1. GitOps 4대 원칙에 포함되지 않는 것은?
1. Declarative
2. Versioned and Immutable
3. **Push로 강제 적용**
4. Continuously Reconciled

**정답: 3.** GitOps는 Pull.

### Q2. ArgoCD의 핵심 reconciliation은 어떤 컴포넌트가 담당?
1. argocd-server
2. **argocd-application-controller**
3. argocd-repo-server
4. argocd-dex

**정답: 2.**

### Q3. Helm vs Kustomize 권장 사용?
1. 외부 패키지 = Helm, 사내 env 분기 = Kustomize
2. 모두 Helm
3. 모두 Kustomize
4. 무관

**정답: 1.**

### Q4. `selfHeal: true`의 의미?
1. 누군가 클러스터 직접 변경해도 Git 정의로 자동 복구
2. ArgoCD 자기 재시작
3. UI 새로고침
4. 무관

**정답: 1.**

### Q5. ApplicationSet의 핵심 가치?
1. 선언적 generator로 N개 Application 동적 생성
2. 단일 Application 수정
3. UI 색상
4. 무관

**정답: 1.**

### Q6. Sync wave의 의미?
1. 리소스 적용 순서 제어
2. UI
3. CPU 우선순위
4. 무관

**정답: 1.**

### Q7. PreSync hook의 대표 용도?
1. DB migration
2. UI 렌더
3. SSO
4. 무관

**정답: 1.**

### Q8. ignoreDifferences가 필요한 대표 케이스?
1. HPA가 다루는 replicas
2. 모든 리소스
3. namespace
4. 무관

**정답: 1.**

### Q9. PullRequest generator의 cleanup 누락 시?
1. 머지된 PR의 preview env가 남아 클러스터 자원 누수
2. 안전
3. 비용 절감
4. 무관

**정답: 1.**

### Q10. SCM Provider generator 토큰의 적절 권한?
1. Read만
2. admin
3. owner
4. 권한 없음

**정답: 1.**

## 섹션 B — 운영 (Q11~Q20)

### Q11. Hub-Spoke의 가장 큰 단점?
1. Hub 장애가 전체 영향
2. UI 분산
3. 느림
4. 무관

**정답: 1.**

### Q12. Controller sharding이 필요해지는 시점?
1. 단일 controller가 reconcile 따라가지 못할 때
2. 시작부터
3. 안 필요
4. UI 느릴 때

**정답: 1.**

### Q13. AppProject의 핵심 가치?
1. 팀별 source/destination/resource 격리
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q14. RBAC에서 allow와 deny 충돌 시?
1. deny가 우선
2. allow가 우선
3. 무작위
4. 순서대로

**정답: 1.**

### Q15. Dex의 역할?
1. SAML/OAuth/LDAP를 OIDC로 브리지
2. RBAC 강제
3. 클러스터 등록
4. UI

**정답: 1.**

### Q16. Webhook 설정의 효과?
1. Git push 즉시 reconcile trigger → polling 지연 제거
2. UI 가속
3. 비용 절감
4. 무관

**정답: 1.**

### Q17. ApplyOutOfSyncOnly=true의 의미?
1. 차이 있는 리소스만 apply
2. 모든 리소스 apply
3. apply 안 함
4. 무관

**정답: 1.**

### Q18. ServerSideApply=true의 활용?
1. 큰 CRD 등에서 client-side apply 한계 우회
2. 모든 경우
3. 무관
4. UI

**정답: 1.**

### Q19. argocd-notifications 트리거 구조?
1. when 조건 + send 템플릿 + subscriptions
2. cron
3. cluster scope
4. 무관

**정답: 1.**

### Q20. Reconcile p95 SLO 빅테크 표준?
1. 30초 ~ 1분
2. 1초
3. 1시간
4. 무관

**정답: 1.**

## 섹션 C — 심화 / 보안 / DR (Q21~Q30)

### Q21. ESO + Vault dynamic secret의 장점?
1. 누출돼도 자동 만료 → 회수 부담 감소
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q22. SealedSecrets의 핵심 약점?
1. controller 키 회전 시 모든 sealed 재암호화
2. 단순
3. UI
4. 무관

**정답: 1.**

### Q23. Image Updater write-back 권장?
1. PR 모드 (사람 리뷰 후 머지)
2. main 직접 commit
3. 무관
4. 의미 없음

**정답: 1.**

### Q24. AnalysisTemplate의 핵심 가치?
1. 메트릭 기반 자동 promotion/abort
2. UI
3. 무관
4. 비용

**정답: 1.**

### Q25. setWeight만으로 트래픽 분할이 정확한가?
1. 아님 — Service Mesh 결합 필요
2. 정확함
3. 무관
4. 의미 없음

**정답: 1.**

### Q26. failureLimit 1의 단점?
1. 일시 jitter도 abort
2. 안전
3. 비용
4. 무관

**정답: 1.**

### Q27. argocd-secret을 git 백업과 별도로 둬야 하는 이유?
1. dex/SSO·repo 개인키 등이 들어 있어 복구 필수
2. 무관
3. 비용
4. UI

**정답: 1.**

### Q28. Active-Active의 잠재 문제?
1. 두 ArgoCD가 같은 앱 reconcile → 충돌
2. 더 안전
3. 비용 절감
4. 무관

**정답: 1.**

### Q29. SOC 2 CC8.1 매핑에서 강력한 증빙?
1. Git history + PR 승인 + ArgoCD sync 기록 통합
2. UI 스크린샷
3. 메모
4. 무관

**정답: 1.**

### Q30. GitOps의 본질적 consistency 모델?
1. Eventual consistency (Git → 클러스터 N초 지연)
2. Strong consistency
3. None
4. 무관

**정답: 1.**

## 채점

- 28~30점: 빅테크 시니어 플랫폼 엔지니어 수준.
- 24~27점: 미드 레벨, DR·컴플라이언스 보완 필요.
- 18~23점: 주니어, RBAC·시크릿·Rollouts 추가 학습.
- ~17점: 핵심 챕터(3, 6, 7, 9주차) 다시 복습 권장.

12주 강의를 끝까지 따라온 여러분, 수고 많으셨습니다. GitOps는 도구가 아니라 **변경관리에 대한 새로운 사고방식**입니다. 진짜 학습은 첫 production 사고를 GitOps로 처리해 본 후에 시작됩니다.
