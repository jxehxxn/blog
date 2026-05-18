---
layout: post
title: "Workflow Week 11: 보안 — Secrets, Sandbox, Supply Chain"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series security
---

## 학습 목표

- Workflow에서 secret 안전 주입.
- Sandbox(gVisor/Kata)로 build job 격리.
- Supply chain 보안(SLSA, in-toto, Cosign).
- 사내 standard practice.

## 1. 비유 — "공장 내 보안 절차"

(1) 도구 보관함 키(secret)는 필요한 직원만, (2) 위험 작업은 격리 부스(sandbox), (3) 모든 제품은 봉인 + 출처 증명(supply chain).

## 2. Secret 주입

### 절대 금지
```yaml
env:
  - { name: DB_PASSWORD, value: "p@ssw0rd" }   # 평문 — 금지
```

### Secret 참조
```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

### ESO + Vault
secret 자체를 K8s에 두지 말고 Vault dynamic. ExternalSecret이 동기화.

### Argo Workflows 표준
```yaml
templates:
  - name: build
    container:
      image: ...
      env:
        - name: SECRET
          valueFrom:
            secretKeyRef:
              name: registry-creds
              key: token
```

## 3. ServiceAccount 분리

각 workflow에 전용 SA. 최소 권한.

```yaml
spec:
  serviceAccountName: argo-build-sa
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: argo-build-sa }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
rules:
  - apiGroups: ["argoproj.io"]
    resources: ["workflows"]
    verbs: ["get", "list"]
```

## 4. Sandbox

Build job은 임의 코드 실행. 컨테이너 escape 위험.

### gVisor (Google)
userspace kernel. 시스템콜 가로채 안전한 subset만.

```yaml
spec:
  runtimeClassName: gvisor
```

### Kata Containers
경량 VM. 강한 격리.

```yaml
spec:
  runtimeClassName: kata
```

빅테크 build farm은 sandbox 표준.

## 5. PSA Restricted

Workflow namespace에 PSA restricted 라벨.

```bash
kubectl label ns argo pod-security.kubernetes.io/enforce=restricted
```

privileged Pod 거부. DinD 같은 위험한 패턴 차단.

## 6. Image admission

cosign으로 서명되지 않은 image는 cluster 진입 불가.

```yaml
# Kyverno policy
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: require-image-signature }
spec:
  validationFailureAction: enforce
  rules:
    - name: check-image
      match:
        resources: { kinds: [Pod] }
      verifyImages:
        - imageReferences: ["mycorp/*"]
          attestors:
            - entries:
                - keyless:
                    issuer: https://accounts.google.com
                    subject: ci-bot@mycorp.iam.gserviceaccount.com
```

Tier 1 Course 6 (OPA/Kyverno) + Course 7 (supply chain) 에서 깊게.

## 7. Network Policy

Workflow Pod이 임의 외부 통신 못 하게:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: workflow-egress }
spec:
  podSelector: { matchLabels: { app: argo-workflows-workflow } }
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector: { matchLabels: { name: kube-system } }
          podSelector: { matchLabels: { k8s-app: kube-dns } }
    - to:
        - ipBlock:
            cidr: 10.0.0.0/8        # 사내 IP만
```

DNS + 사내만 허용. 외부 데이터 유출 차단.

## 8. SLSA (Supply chain Levels for Software Artifacts)

빅테크 표준. 4 레벨:
- Level 1: 빌드 스크립트 공개.
- Level 2: 빌드 서비스 사용.
- Level 3: 빌드 격리 + provenance 서명.
- Level 4: 2인 리뷰 + 재현 가능 빌드.

Tekton Chains가 Level 3까지 자동.

## 9. Audit

모든 workflow 실행을 K8s audit + SIEM 적재:
- 누가 submit
- 어떤 source repo / commit
- 어떤 image 빌드 / push
- 어떤 secret 사용

7년 보존 (SOC 2 표준).

## 10. 운영 함정 5선

1. SA 권한 과다 → workflow가 cluster admin.
2. privileged Pod 사용 → escape 가능.
3. Untrusted source 빌드 → 악성 코드 주입.
4. Secret 노출 (env, log).
5. Audit 누락.

## 11. 실습

```bash
# 1. ESO + Vault로 secret 주입
# 2. PSA restricted namespace에서 workflow 실행
# 3. gVisor runtime class 적용
# 4. Kyverno로 image signature 강제
# 5. NetworkPolicy로 외부 egress 차단
```

## 12. 자가평가 퀴즈

### Q1. Secret 절대 금지?
1. **env value에 평문**
2. secretRef
3. ESO
4. Vault

**정답: 1.**

### Q2. Sandbox 사용 시점?
1. **untrusted 코드 빌드 (PR build 등)**
2. 모든 경우
3. 무관
4. UI

**정답: 1.**

### Q3. SLSA Level 3 의 핵심?
1. **빌드 격리 + provenance 서명**
2. UI
3. 무료
4. 무관

**정답: 1.**

### Q4. Network Policy로 build Pod 제한?
1. **외부 데이터 유출 차단**
2. UI
3. 비용
4. 무관

**정답: 1.**

### Q5. Audit 보존 권장?
1. **7년 (SOC 2 등)**
2. 1일
3. 1주
4. 무관

**정답: 1.**

## 13. 다음 주차

[Week 12: 캡스톤]에서 풀스택 CI/CD 플랫폼 구축.
