---
layout: post
title: "ArgoCD Week 2: 설치부터 첫 sync까지, 그리고 모든 컴포넌트 한 줄도 빠짐없이 읽기"
date: 2026-05-18 14:30:00 +0900
categories: devops gitops argocd kubernetes
---

## 학습 목표

- ArgoCD를 3가지 방식(manifest/Helm/Operator)으로 설치한다.
- 첫 Application을 만들어 sync한다.
- ArgoCD의 5대 컴포넌트(API server, repo server, application controller, dex, redis) 책임을 안다.
- `argocd` CLI의 핵심 명령을 손에 익힌다.

## 1. 설치 — 3가지 경로

### (a) 공식 manifest (가장 빠름, 학습용)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/v2.11.0/manifests/install.yaml
```

기본 admin 패스워드 확인:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### (b) Helm chart (production 시작점)

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace \
  --set server.ingress.enabled=true \
  --set server.ingress.hostname=argocd.mycorp.com
```

### (c) ArgoCD Operator (멀티 인스턴스/멀티 테넌트)

OperatorHub의 ArgoCD Operator를 통해 ArgoCD 인스턴스를 CR로 관리. 빅테크 멀티팀 환경에서 사용.

## 2. CLI 로그인 + 첫 Application

```bash
argocd login argocd.mycorp.com
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
argocd app sync guestbook
argocd app wait guestbook --health
```

UI에서 확인하면 Application 카드가 등장하고, Health/Sync 상태가 표시됩니다.

## 3. 5대 컴포넌트 — 한 줄도 빠짐없이

ArgoCD는 단일 바이너리가 아니라 마이크로서비스입니다.

### (a) `argocd-server` (API server)
- gRPC + REST API + Web UI.
- 권한 검증, SSO 종착점, ApplicationSet/Application CRUD.
- 무상태(stateless), HPA 가능.

### (b) `argocd-application-controller` (핵심 reconciliation)
- Kubernetes operator. Application CR을 감시하고 desired vs live 비교, sync 실행.
- 가장 많은 CPU/메모리 사용. 클러스터 수·앱 수에 따라 sharding.
- `--app-resync` 주기로 polling (기본 180초).

### (c) `argocd-repo-server` (Git 처리)
- Git clone, Helm render, Kustomize build, plugin 실행.
- 빠르게 stateless로 스케일 가능. CPU bound.
- 외부 인터넷·git 접근 책임 분리.

### (d) `argocd-dex-server` (SSO)
- OIDC 브리지. SAML/OAuth/LDAP를 OIDC로 변환해 server에 제공.
- 7주차에서 다룸.

### (e) `argocd-redis` (캐시)
- repo 결과·세션·throttling 캐시.
- HA 모드에서는 redis-ha 또는 외부 redis 권장.

## 4. Application 한 줄 한 줄 읽기

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default              # 7주차에서 다룰 Project
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD        # branch/tag/commit
    path: guestbook
    # helm 또는 kustomize 블록을 여기 추가 (5주차)
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true               # 삭제된 리소스 정리
      selfHeal: true            # 수동 변경 자동 복구
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

각 필드를 외워두면 4주차 sync 전략이 훨씬 쉽습니다.

## 5. CLI 핵심 명령

```bash
argocd app list                        # 전체 앱
argocd app get guestbook               # 상세
argocd app diff guestbook              # desired vs live diff
argocd app sync guestbook --dry-run
argocd app history guestbook
argocd app rollback guestbook <id>
argocd app delete guestbook
argocd login --sso                     # SSO (7주차)
argocd cluster add mycluster           # 멀티클러스터 (6주차)
argocd repo add ... --ssh-private-key  # 리포 등록
```

## 6. UI 핵심 화면

- Applications: 카드 그리드, health(녹/노/적)와 sync 상태 색.
- Application 상세: tree view로 모든 K8s 리소스가 한 화면에.
- Sync history: 누가 언제 어떤 commit으로 sync했는지.
- Settings: cluster, repo, project, RBAC.

빅테크 운영 팁: **Application 상세 tree view는 디버깅의 시작**. 어떤 리소스가 OutOfSync인지 즉시 보입니다.

## 7. 실습 과제

1. 위 manifest로 로컬 kind 클러스터에 ArgoCD 설치.
2. argocd-example-apps의 다른 디렉토리 3개를 각각 Application으로 등록.
3. 의도적으로 `kubectl edit deployment` 로 spec을 변경 후 ArgoCD가 self-heal로 복구하는 것을 30초 안에 확인.
4. CLI로 history 조회 → 첫 rollback 실험.
5. 결과를 1쪽으로 정리.

## 8. 자가평가 퀴즈

### Q1. `--app-resync` 기본값의 의미는?
1. ArgoCD가 desired state와 실제 상태를 비교·동기화하는 주기
2. UI 새로고침 주기
3. SSO 토큰 갱신 주기
4. Redis flush 주기

**정답: 1.**

### Q2. `repo-server`의 핵심 역할은?
1. Git 클론, Helm/Kustomize 렌더링
2. UI 호스팅
3. 클러스터 reconciliation
4. SSO

**정답: 1.**

### Q3. `selfHeal: true`의 의미는?
1. 누군가 클러스터를 직접 변경해도 Git 정의로 자동 복구
2. ArgoCD가 자기 자신을 재시작
3. Redis 자동 복구
4. 의미 없음

**정답: 1.**

### Q4. `prune: true` 누락 시 일어나는 일은?
1. Git에서 삭제된 리소스가 클러스터에 잔존(orphan)
2. Sync 실패
3. UI에 표시 안 됨
4. 정상

**정답: 1.**

### Q5. Application 상세 tree view가 디버깅에 유용한 이유?
1. 어떤 리소스가 OutOfSync인지 시각적으로 즉시 식별
2. 색이 예뻐서
3. CPU 사용량 표시
4. 무관

**정답: 1.**

## 9. 다음 주차

[Week 3: Application / ApplicationSet]에서는 단일 Application의 한계를 만나고, ApplicationSet 8종 generator로 1,000개 앱을 한 번에 다루는 패턴을 익힙니다.
