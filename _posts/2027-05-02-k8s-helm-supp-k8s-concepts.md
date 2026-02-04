---
layout: post
title: "Kubernetes Package Management with Helm: Supplement - Kubernetes Core Concepts Refresher"
---

Week 1 과제로 쿠버네티스 리소스 복습을 드렸습니다.
"다 아는 내용인데?" 싶을 수도 있지만, Helm 템플릿을 짤 때 가장 많이 실수하는 부분이 바로 이 **기본 리소스의 스펙(Spec)**입니다.

Helm은 결국 **"쿠버네티스 YAML을 생성하는 기계"**일 뿐입니다. 생성된 YAML이 문법에 안 맞으면 쿠버네티스는 거부합니다. 따라서 쿠버네티스 리소스를 정확히 이해하는 것이 Helm의 전제조건입니다.

핵심만 빠르게 짚고 넘어갑시다.

---

## 1. Workloads

### Pod (파드)
*   **정의:** 쿠버네티스에서 배포할 수 있는 가장 작은 단위.
*   **특징:** 하나 이상의 컨테이너가 들어감. IP 하나를 공유함. 죽으면 끝(Ephemeral).
*   **Helm 포인트:** 템플릿 작성 시 `image`, `command`, `args`, `env` 부분이 가장 많이 변수화됩니다.

### Deployment (디플로이먼트)
*   **정의:** Pod의 복제본(Replica)을 관리하고 배포(Rollout/Rollback)를 담당하는 컨트롤러.
*   **역할:** "웹 서버 파드 3개 유지해. 버전 2로 업데이트할 때는 하나씩 갈아끼워."
*   **StatefulSet과의 차이:** Deployment는 파드끼리 구분 안 함(개미 1, 2, 3). StatefulSet은 순서와 이름이 있음(개미-0, 개미-1). DB 같은 상태 저장 앱은 StatefulSet을 씁니다.

---

## 2. Networking

### Service (서비스)
*   **정의:** 파드들은 자꾸 죽고 IP가 바뀝니다. 서비스는 고정된 IP나 DNS 이름으로 파드들에게 트래픽을 중계해줍니다.
*   **Type:**
    *   **ClusterIP:** 내부용. (기본값)
    *   **NodePort:** 외부용. 모든 노드의 특정 포트를 엽니다. (잘 안 씀)
    *   **LoadBalancer:** 외부용. 클라우드(AWS/GCP)의 로드밸런서를 붙여줍니다.

### Ingress (인그레스)
*   **정의:** HTTP/HTTPS 라우터. "domain.com/api는 서비스 A로, /web은 서비스 B로 보내라."
*   **특징:** Service(LoadBalancer) 하나만 뚫어놓고(Nginx Ingress Controller), 그 뒤에서 도메인 기반 라우팅을 수행합니다. 비용 절감의 핵심입니다.

---

## 3. Configuration

### ConfigMap & Secret
*   **ConfigMap:** 설정 파일, 환경 변수 등 비민감 정보.
*   **Secret:** 비밀번호, 인증서 등 민감 정보. (Base64 인코딩됨 - 암호화는 아님 주의!)
*   **Helm 포인트:** `values.yaml`에 있는 설정값들이 렌더링되어 ConfigMap으로 들어갑니다. 파드가 이걸 마운트해서 앱 설정으로 씁니다.

---

## 4. Helm 개발자가 자주 틀리는 Spec (Gotchas)

1.  **Labels & Selectors:**
    *   Deployment의 `.spec.selector.matchLabels`와 Pod Template의 `.metadata.labels`는 **반드시 일치**해야 합니다. 오타 나면 배포 안 됩니다.
    *   Helm의 `_helpers.tpl`을 써서 공통 라벨을 관리하는 이유입니다.

2.  **Ports:**
    *   ContainerPort (파드 내부) -> Service TargetPort (파드 쪽) -> Service Port (서비스 노출) 이 연결 고리가 끊기면 통신 불능.

3.  **Resources (CPU/Mem):**
    *   `limits`와 `requests`를 혼동하지 마세요.
    *   Helm 템플릿에서 `{{ .Values.resources | toYaml }}` 같은 함수를 써서 통째로 넘기는 패턴을 많이 씁니다.

이 기초가 탄탄해야 다음 주부터 시작될 템플릿 작성이 즐거워집니다.
