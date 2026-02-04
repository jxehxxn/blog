---
layout: post
title: "Kubernetes Package Management with Helm: Week 12 - Capstone: Production Grade Deployment"
---

12주간의 여정이 끝났습니다.
단순한 YAML 파일 관리자였던 Helm이, 이제는 여러분 손에서 **완전한 배포 플랫폼**이 되었습니다.

마지막 과제는 여러분이 배운 모든 기술을 총동원하여, 실제 기업 환경과 동일한 **Production Grade MSA Architecture**를 구축하는 것입니다.

---

## 🏆 Capstone Project: "E-Commerce Microservices Stack"

### 1. Requirements (요구사항)

가상의 쇼핑몰 시스템을 구축합니다.

**Microservices (Subcharts):**
1.  **Frontend:** React/Vue 앱 (Nginx).
2.  **Product Service:** 상품 목록 제공 (Node.js/Go).
3.  **Cart Service:** 장바구니 (Redis 의존성).
4.  **Database:** MySQL 또는 MongoDB (의존성 차트 활용).

**Infrastructure (Umbrella Chart):**
*   이 모든 서비스를 하나의 `shop-umbrella` 차트로 묶어서 배포해야 합니다.

**Observability:**
*   모든 서비스는 `/metrics`를 노출하고 `ServiceMonitor`로 수집되어야 합니다.
*   모든 로그는 Loki로 수집되어야 합니다. (에러 발생 시 검색 가능해야 함)
*   Frontend -> Product -> DB 로 이어지는 호출 트레이스가 Jaeger에 나와야 합니다.

**Automation:**
*   Skaffold로 로컬 개발 환경이 구성되어야 합니다. (`skaffold dev`)
*   Ingress를 통해 `shop.example.com` 도메인으로 접속 가능해야 하며, HTTPS(Self-signed OK)가 적용되어야 합니다.

---

## 2. Architecture Guidelines

*   **Structure:**
    ```
    shop-umbrella/
    ├── Chart.yaml
    ├── values.yaml
    ├── charts/
    │   ├── frontend/
    │   ├── product/
    │   └── cart/
    └── templates/
        └── ingress.yaml (통합 인그레스)
    ```
*   **Networking:** 프론트엔드는 백엔드 서비스를 `http://localhost`가 아닌 `http://product-service` (K8s Service DNS)로 호출해야 합니다. 이 주소를 `values.yaml`에서 주입하세요.

---

## 3. Submission Deliverables

1.  **GitHub Repository:** 모든 소스 코드와 차트 파일.
2.  **Demo Video (3분):**
    *   `helm install` 한 방에 모든 파드가 뜨는 모습.
    *   웹 브라우저에서 쇼핑몰 접속 및 상품 담기 시연.
    *   Grafana 대시보드에서 RPS 그래프 확인.
    *   Jaeger에서 트레이스 확인.
3.  **Post-Mortem:** 개발 중 겪었던 가장 어려웠던 문제(Trouble)와 해결 과정(Shooting) 1가지 서술.

---

## 🎓 Closing Remarks

30년 전, 저는 서버 한 대에 아파치를 띄우고 "배포했다"고 말했습니다.
지금 여러분은 수십 개의 마이크로서비스를, 수백 개의 컨테이너를, 코드 몇 줄로 지휘하고 있습니다.

이 힘(Helm)을 올바르게 쓰십시오.
복잡함을 단순함으로 감싸되(Abstraction), 그 내부의 원리(Understanding)는 잊지 마십시오.

여러분은 이제 **Kubernetes Native Architect**입니다.
수고하셨습니다.

**Helm install success.**
