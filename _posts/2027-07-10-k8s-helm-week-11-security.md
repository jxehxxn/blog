---
layout: post
title: "Kubernetes Package Management with Helm: Week 11 - Security & Certificate Management (Cert-Manager)"
---

브라우저 주소창에 "주의 요함(Not Secure)" 빨간 딱지가 붙어있으면 아무도 그 서비스를 쓰지 않습니다.
HTTPS는 필수입니다. 하지만 SSL 인증서는 비싸고, 90일마다 갱신해줘야 합니다. 까먹으면 서비스 장애로 이어지죠.

**Cert-Manager**는 쿠버네티스 네이티브한 방식으로 인증서를 **자동 발급, 자동 갱신, 자동 적용**해줍니다. Helm 차트에 단 4줄만 추가하면 됩니다.

---

## 1. How Cert-Manager Works

1.  **Issuer (발급자):** "누구한테 인증서를 받을래?" (Let's Encrypt, SelfSigned, Vault 등)
2.  **Certificate (인증서):** "a.com 인증서 줘."
3.  **Challenge (검증):** "너 a.com 주인 맞아? 이 파일 올려봐(HTTP-01) 혹은 DNS 레코드 추가해봐(DNS-01)."
4.  **Secret:** 검증 통과 시 `tls-secret`이라는 이름의 Secret 생성.

---

## 2. Helm Integration: Annotations Magic

Ingress 리소스에 Annotation 몇 줄만 붙이면 Cert-Manager가 알아서 동작합니다.

```yaml
# templates/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    # 1. ClusterIssuer 지정 (Let's Encrypt Prod)
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    # 2. ACME Challenge 방식 (보통 http01)
    kubernetes.io/tls-acme: "true"
spec:
  tls:
    - hosts:
        - {{ .Values.ingress.host }}
      secretName: {{ .Release.Name }}-tls # 여기에 인증서가 저장됨
```

이게 끝입니다. 나머지는 Cert-Manager가 알아서 합니다.

---

## 3. DNS-01 Challenge (Wildcard Cert)

`*.example.com` 같은 와일드카드 인증서를 받으려면 HTTP 방식으론 안 됩니다. DNS 레코드를 수정할 수 있어야 합니다.
이때는 AWS Route53이나 Google Cloud DNS와 연동해야 합니다.

Cert-Manager Helm 차트를 설치할 때 Cloud Provider 자격 증명(ServiceAccount)을 넣어줘야 합니다.

---

## 🛠️ Lab: Self-Signed Certificate

로컬 환경(Minikube/Kind)에서는 Let's Encrypt를 쓰기 어렵습니다(공인 IP가 없으니까요).
대신 자체 서명(Self-Signed) 인증서로 HTTPS를 흉내내 봅니다.

1.  **Cert-Manager 설치:** `helm install cert-manager jetstack/cert-manager --set installCRDs=true`
2.  **SelfSigned Issuer 생성:**
    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: selfsigned
    spec:
      selfSigned: {}
    ```
3.  **Ingress 수정:** `cert-manager.io/cluster-issuer: "selfsigned"`
4.  **확인:** 브라우저 접속 시 "안전하지 않음" 경고가 뜨지만, 인증서 정보를 보면 `selfsigned` 발급자가 보입니다. HTTPS 적용 성공!

---

## 📝 11주차 과제: Let's Encrypt Staging

**목표:** 실제 공인 도메인이 없다면 `nip.io`를 사용해서라도 Let's Encrypt Staging 환경에서 인증서를 발급받으세요.

1.  Let's Encrypt Staging 서버를 사용하는 `ClusterIssuer`를 만듭니다. (이메일 주소 필수)
2.  Week 10에서 만든 Ingress에 적용합니다.
3.  `kubectl get certificate` 명령어로 상태가 `Ready: True`가 되는지 확인합니다.
4.  `kubectl describe order` 등을 통해 발급 과정을 추적한 로그를 제출하세요.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** Let's Encrypt 인증서의 유효 기간은 며칠인가요? (90일)
2.  **Q2.** 와일드카드 인증서(`*.example.com`)를 발급받기 위해 반드시 사용해야 하는 검증 방식(Challenge Type)은? (DNS-01)
3.  **Q3.** Cert-Manager에서 발급받은 인증서와 개인키는 쿠버네티스의 어떤 리소스 타입으로 저장되나요? (Secret - `kubernetes.io/tls` 타입)

---

보안까지 챙겼습니다. 이제 12주차, 대망의 **Capstone Project**만 남았습니다.
지금까지 배운 모든 것(Helm, Skaffold, Prometheus, Loki, Jaeger, Ingress, Cert-Manager)을 하나의 거대한 파이프라인으로 통합할 시간입니다.

**Encrypt All The Things.**
