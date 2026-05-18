---
layout: post
title: "SC Week 7: Tekton Chains 운영 깊게"
date: 2026-05-18 21:00:00 +0900
categories: security supply-chain tekton chains platform senior-series
---

## 학습 목표

- Tekton Chains 설치 + 설정.
- Provenance/SBOM/sig 자동 생성.
- 운영 메트릭.

## 1. Chains의 책임

PipelineRun/TaskRun 완료 → 자동:
1. 산출물 식별.
2. SLSA provenance 생성.
3. cosign 서명.
4. OCI + Rekor 적재.

## 2. 설치

```bash
kubectl apply -f https://storage.googleapis.com/tekton-releases/chains/latest/release.yaml
```

## 3. 핵심 설정

```bash
kubectl patch configmap chains-config -n tekton-chains --type merge -p '{
  "data": {
    "artifacts.taskrun.format": "in-toto",
    "artifacts.taskrun.storage": "oci",
    "artifacts.taskrun.signer": "x509",
    "artifacts.pipelinerun.format": "in-toto",
    "artifacts.pipelinerun.storage": "oci",
    "artifacts.pipelinerun.signer": "x509",
    "transparency.enabled": "true",
    "transparency.url": "https://rekor.sigstore.dev"
  }
}'
```

## 4. Signing Key

### Static
```bash
cosign generate-key-pair k8s://tekton-chains/signing-secrets
```

### KMS (권장)
```bash
# data: { "signers.x509.fulcio.enabled": "true" }
```

Fulcio + GCP/AWS KMS.

## 5. Provenance 자동

PipelineRun이 다음 결과 출력:
- `IMAGE_URL`: 빌드된 image.
- `IMAGE_DIGEST`: 해시.

Chains가 이를 식별해 provenance 생성.

```yaml
spec:
  results:
    - name: IMAGE_URL
    - name: IMAGE_DIGEST
```

## 6. SBOM 통합

```yaml
spec:
  results:
    - name: IMAGES
    - name: SBOM
```

Chains가 SBOM도 attest.

## 7. 검증

```bash
cosign verify <image> --key cosign.pub
cosign verify-attestation --type slsaprovenance --key cosign.pub <image>
```

Kyverno admission으로 cluster 진입 차단.

## 8. 모니터링

```promql
chains_signed_total
chains_payload_uploaded_total
chains_signed_failed_total
```

서명 fail 알람.

## 9. 운영 함정

1. controller 메모리 부족 → 큰 PipelineRun에서 OOM.
2. Rekor 접속 실패 → 서명 실패.
3. KMS rate limit.
4. private Rekor와 public 혼용.
5. SBOM 생성 누락.

## 10. 자가평가

### Q1. Chains 자동 책임?
1. **provenance + 서명 + Rekor + OCI 적재**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q2. 표준 결과 이름?
1. **IMAGE_URL + IMAGE_DIGEST**
2. result 3. 무관 4. UI

**정답: 1.**

### Q3. Signing 권장?
1. **KMS + Fulcio**
2. static 3. UI 4. 무관

**정답: 1.**

### Q4. transparency 활성?
1. **Rekor 적재 → audit ledger**
2. UI 3. 빠름 4. 무관

**정답: 1.**

### Q5. 검증 통합?
1. **Kyverno admission**
2. UI 3. 빠름 4. 무관

**정답: 1.**

## 11. 다음

[Week 8: Admission verify].
