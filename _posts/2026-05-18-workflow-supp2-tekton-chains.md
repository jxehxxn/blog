---
layout: post
title: "Workflow 보충 2: Tekton Chains + 공급망 보안 깊게"
date: 2026-05-18 17:00:00 +0900
categories: cicd workflow argo-workflows tekton platform senior-series supplement
---

7주차 Chains을 SLSA·in-toto 수준으로 풉니다.

## 1. SLSA — 4 레벨 요구

| Level | 요구 |
|-------|------|
| 1 | Provenance available |
| 2 | Tampering resistance of build service |
| 3 | Hosted build platform + non-falsifiable provenance |
| 4 | Two-person review + hermetic build + reproducible |

Tekton Chains가 Level 2~3을 자동.

## 2. Provenance 란

빌드 과정의 메타데이터:
- source repo + commit SHA
- builder identity
- build instructions
- materials (dependencies)
- result artifact digest

JSON 형식, in-toto 표준.

## 3. Chains 동작

```
[TaskRun 완료]
   ↓
[Chains controller가 결과 관측]
   ↓
[provenance 생성 (in-toto SLSA v1.0)]
   ↓
[cosign으로 서명 (private key 또는 keyless)]
   ↓
[OCI registry에 signature + attestation 저장]
   ↓
[Rekor (transparency log) 적재]
```

## 4. 설치 + 설정

```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/chains/latest/release.yaml

# 설정
kubectl patch configmap chains-config -n tekton-chains --type merge -p '{"data": {
  "artifacts.taskrun.format": "in-toto",
  "artifacts.taskrun.storage": "oci",
  "artifacts.taskrun.signer": "x509",
  "transparency.enabled": "true",
  "transparency.url": "https://rekor.sigstore.dev"
}}'
```

## 5. Cosign 서명 방식

### Static key
```bash
cosign generate-key-pair k8s://tekton-chains/signing-secrets
# secret 생성
```

### Keyless (Sigstore)
OIDC token (GitHub Actions, K8s SA projected token) 으로 일회용 서명. 정적 key 관리 부담 0.

빅테크 점점 keyless 표준.

## 6. Verification

```bash
cosign verify --key cosign.pub <image>
cosign verify-attestation --type slsaprovenance --key cosign.pub <image>
```

policy in admission:
```yaml
# Kyverno
spec:
  rules:
    - name: verify
      verifyImages:
        - imageReferences: ["mycorp/*"]
          attestations:
            - predicateType: https://slsa.dev/provenance/v1.0
              attestors:
                - entries:
                    - keyless:
                        issuer: https://token.actions.githubusercontent.com
                        subject: mycorp/repo
```

## 7. 빅테크 사례 — Google

Google 사내 모든 build에 Chains. SLSA Level 3 표준. policy로 unsigned image cluster 진입 차단.

## 8. 운영 함정

1. Chains 미설정 → evidence 없음.
2. private key 분실 → 검증 불가.
3. keyless OIDC 잘못 → 잘못된 issuer 신뢰.
4. Rekor 미사용 → transparency log 부재.
5. admission policy 미설정 → 서명만 하고 검증 안 함.

## 9. 결론

Tekton Chains는 SLSA를 자동화하는 빅테크 표준. 시니어는 빌드부터 admission까지 풀스택 supply chain을 설계해야 합니다. 자세한 영역은 Tier 1 Course 7.
