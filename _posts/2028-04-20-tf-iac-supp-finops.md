---
layout: post
title: "Infrastructure as Code with Terraform: Supplement - FinOps & Cost Estimation (Infracost)"
---

"이거 배포하면 한 달에 얼마 나와요?"
클라우드 엔지니어가 가장 많이 듣지만, 가장 대답하기 어려운 질문입니다.

AWS 계산기(Calculator)를 두드리는 건 구식입니다.
우리는 **코드 레벨에서 비용을 예측**하고, 예산을 초과하면 배포를 막는 **FinOps**를 구현할 것입니다.

---

## 1. Why Cost Estimation in IaC?

클라우드 비용은 "고지서를 받아봐야 아는" 것이 아닙니다.
코드를 짜는 순간 비용은 확정됩니다.

*   `instance_type = "t3.micro"` -> 월 $7
*   `instance_type = "p4d.24xlarge"` -> 월 $23,000

실수로 인스턴스 타입을 잘못 적어서 수천만 원이 나오는 사고(Bill Shock)를 막으려면, **PR 단계에서 비용을 검사**해야 합니다.

---

## 2. Infracost: The Standard Tool

Infracost는 Terraform 코드를 분석하여 예상 비용을 계산해주는 오픈소스 도구입니다.

### How it works
1.  Terraform 코드를 파싱합니다.
2.  사용된 리소스(EC2, RDS 등)를 식별합니다.
3.  Cloud Vendor(AWS)의 최신 가격 API를 조회합니다.
4.  "이 변경 사항은 월 $50를 추가합니다"라고 리포트를 줍니다.

---

## 3. Integration with GitHub Actions

PR을 올리면 댓글로 견적서를 붙여줍니다.

```yaml
# .github/workflows/infracost.yml
jobs:
  infracost:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Infracost
        uses: infracost/actions/setup@v2
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Generate breakdown
        run: infracost breakdown --path . --format json --out-file infracost.json

      - name: Post comment
        uses: infracost/actions/comment@v2
        with:
          path: infracost.json
          behavior: update
```

---

## 4. Advanced Cost Policies (OPA)

"월 $500 이상 증가하는 PR은 팀장 승인이 필요하다."
단순히 보여주는 걸 넘어, 정책으로 강제할 수 있습니다.

Infracost는 JSON 출력을 제공하므로, **OPA(Open Policy Agent)**와 결합할 수 있습니다.

```rego
# cost-policy.rego
deny[msg] {
  input.diffTotalMonthlyCost > 500
  msg := sprintf("Cost increase ($%.2f) exceeds limit ($500)", [input.diffTotalMonthlyCost])
}
```

---

## 🛠️ Lab: Preventing Bill Shock

1.  **설치:** `brew install infracost`
2.  **API Key:** `infracost auth login`
3.  **코드 작성:** `main.tf`에 비싼 리소스(`r5.8xlarge`)를 추가해봅니다.
4.  **견적 확인:** `infracost breakdown --path .`
    *   터미널에 상세한 비용 내역이 출력되는지 확인하세요.
5.  **Usage File:** S3나 Lambda는 사용량(요청 횟수)에 따라 비용이 다릅니다. `infracost-usage.yml` 파일을 만들어 예상 트래픽을 입력하고 다시 계산해봅니다.

---

## 5. Summary

*   **Shift Left:** 비용 관리를 고지서 받는 시점이 아니라, 코드를 짜는 시점(Left)으로 당겨옵니다.
*   **Visibility:** 개발자가 자신이 만드는 리소스의 가격을 알게 되면, 자연스럽게 아껴 쓰게 됩니다.

이제 여러분은 인프라 아키텍트이자 **CFO(재무 이사)**의 역할도 수행할 수 있습니다.
