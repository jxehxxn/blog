---
layout: post
title: "Infrastructure as Code with Terraform: Supplement - Secrets Management with HashiCorp Vault"
---

`terraform.tfstate` 파일은 평문(Plaintext)입니다.
여기에 RDS 비밀번호가 적혀있다면? 그 파일을 열람할 수 있는 모든 사람이 DB를 털 수 있습니다.

**"비밀은 코드에 적지 않는다. State에도 남기지 않는다."**
이것이 보안의 철칙입니다. 이를 위해 **HashiCorp Vault**를 Terraform과 통합하는 방법을 배웁니다.

---

## 1. The Problem of Secrets in Terraform

Terraform으로 DB를 만들 때 비밀번호를 어떻게 넣나요?

1.  **하드코딩:** `password = "1234"` (최악)
2.  **변수 (`terraform.tfvars`):** 파일이 유출되면 끝.
3.  **환경변수 (`TF_VAR_db_password`):** 그나마 낫지만, 여전히 **State 파일에는 평문으로 저장**됩니다.

---

## 2. Vault Provider: The Dynamic Secrets

Vault는 비밀을 저장만 하는 금고가 아닙니다. **비밀을 찍어내는 공장**입니다.

### Dynamic Secrets (동적 비밀)
Terraform이 Vault에게 "AWS 접속하게 키 줘"라고 요청하면, Vault는 그 즉시 **유효기간 10분짜리 임시 Access Key**를 발급해서 줍니다.
Terraform은 이 키로 인프라를 만들고, 10분 뒤 키는 자동 파기됩니다.
유출될 비밀 자체가 사라지는 것입니다.

---

## 3. Integrating Vault with Terraform

### Step 1: Vault Provider Setup
```hcl
provider "vault" {
  address = "https://vault.example.com"
}
```

### Step 2: Reading Secrets
DB 비밀번호를 Vault의 `secret/db` 경로에서 읽어옵니다.
```hcl
data "vault_generic_secret" "db" {
  path = "secret/db"
}

resource "aws_db_instance" "default" {
  password = data.vault_generic_secret.db.data["password"]
}
```
**주의:** 이렇게 해도 State 파일에는 여전히 남습니다.

### Step 3: The Ultimate Solution (Terraform Cloud Agents)
완벽한 보안을 위해서는 **Terraform 코드가 실행되는 환경 자체를 격리**해야 합니다. Vault가 주입해준 토큰을 메모리에서만 쓰고 버리는 방식입니다.

---

## 4. Vault-Backed State Encryption

State 파일 자체를 암호화할 수도 있습니다.
S3 Backend를 쓸 때, KMS 키 대신 Vault의 **Transit Engine**을 사용하여 암호화/복호화를 수행할 수 있습니다.

---

## 🛠️ Lab: Injecting Secrets

로컬에 Vault를 띄우고 Terraform에서 비밀을 가져와 봅니다.

1.  **Vault 실행:** `vault server -dev`
2.  **비밀 저장:** `vault kv put secret/db password=supersecret`
3.  **Terraform 코드:** `data "vault_generic_secret"`을 사용하여 위 비밀을 읽어와서 `output`으로 출력해봅니다. (Sensitive 처리 필수)
    ```hcl
    output "db_pass" {
      value     = data.vault_generic_secret.db.data["password"]
      sensitive = true # 콘솔에 출력되지 않게!
    }
    ```

---

## 5. Summary

*   **Static Secrets:** Vault에 저장된 고정된 비밀번호를 읽어옴.
*   **Dynamic Secrets:** 그때그때 생성되는 임시 자격 증명 사용. (권장)
*   **Encryption:** State 파일 자체를 암호화.

보안은 불편합니다. 하지만 털리는 것보다는 낫습니다.
Vault를 마스터하면 보안 팀의 사랑을 독차지할 수 있습니다.
