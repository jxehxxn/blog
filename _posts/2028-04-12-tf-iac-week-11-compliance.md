---
layout: post
title: "Infrastructure as Code with Terraform: Week 11 - Security & Compliance as Code"
---

"개발자가 실수로 0.0.0.0/0을 열었어요."
"누가 비싼 GPU 인스턴스를 띄웠어요."

이런 실수는 사람을 혼낸다고 해결되지 않습니다. **시스템이 막아야 합니다.**
Policy as Code(PaC)는 인프라가 배포되기 전에 규정 위반을 잡아냅니다.

---

## 1. HashiCorp Sentinel

Terraform Enterprise/Cloud 전용 정책 엔진입니다.
*   **언어:** Sentinel Language (사람이 읽기 쉬움).
*   **기능:** `hard-mandatory` (절대 불가), `soft-mandatory` (승인 시 가능), `advisory` (경고).

```sentinel
# EC2 인스턴스 타입 제한 정책
allowed_types = ["t2.micro", "t2.small"]

main = rule {
  all tfplan.resource_changes as _, rc {
    rc.type is "aws_instance" implies
      rc.change.after.instance_type in allowed_types
  }
}
```

---

## 2. OPA (Open Policy Agent) Revisited

Week 8에서 잠깐 봤던 OPA는 오픈소스 진영의 표준입니다.
CI/CD 파이프라인(`conftest`)에서 돌리거나, Kubernetes Admission Controller(`Gatekeeper`)로 쓸 수 있습니다.

---

## 3. Drift Detection (구성 변경 탐지)

누가 코드를 안 거치고 콘솔에서 몰래 포트를 열었다면?
Terraform State와 실제 인프라가 달라집니다. 이를 **Drift**라고 합니다.

*   **해결:** 매일 밤 `terraform plan`을 스케줄링(Cron)합니다.
*   **알림:** "변경 사항이 감지됨! (Changes detected)"라고 슬랙에 뜹니다.
*   **조치:** `terraform apply`를 해서 코드로 덮어씌우거나(원상복구), 코드를 수정해서 반영합니다.

---

## 🛠️ Lab: Drift Detection Bot

1.  GitHub Actions에 `schedule` (Cron) 트리거를 추가합니다. (`cron: '0 0 * * *'`)
2.  `terraform plan -detailed-exitcode`를 실행합니다.
    *   Exit Code 0: 변경 없음 (Succeeded).
    *   Exit Code 2: 변경 있음 (Diff).
3.  Exit Code 2일 때 슬랙으로 웹훅을 쏘는 스텝을 추가합니다.
4.  콘솔에서 태그 하나를 몰래 바꾸고 다음 날(혹은 수동 실행) 알림이 오는지 봅니다.

---

## 📝 11주차 과제: Compliance Rules

**목표:** 다음 보안 규정을 **OPA(Rego)** 코드로 작성하세요.

1.  **Rule 1:** 모든 S3 버킷은 `versioning`이 켜져 있어야 한다.
2.  **Rule 2:** 모든 EBS 볼륨은 `encrypted: true`여야 한다.
3.  **Rule 3:** RDS 인스턴스는 `publicly_accessible: false`여야 한다.

---

## ✅ Self-Assessment Quiz

1.  **Q1.** HashiCorp Sentinel에서 정책 위반 시 배포를 무조건 차단하는 강제성 레벨(Enforcement Level)은? (`hard-mandatory`)
2.  **Q2.** 코드를 통하지 않고 수동으로 인프라를 변경하여 State 파일과 실제 형상이 달라지는 현상을 무엇이라 하나요? (Drift / Configuration Drift)
3.  **Q3.** `terraform plan` 명령어 실행 시 변경 사항이 있을 때 0이 아닌 특정 종료 코드(Exit Code)를 반환하게 하는 플래그는? (`-detailed-exitcode` -> 반환값 2)

---

이제 대망의 마지막 주차입니다.
지금까지 배운 모든 것(Module, IAM, Network, Compute, CI/CD, Security)을 집대성하여 **엔터프라이즈급 인프라**를 구축합니다.

**Secure by Design.**
