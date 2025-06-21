

##  Jenkins + Vault + Terraform Integration: Overview

###  Goal:

Securely retrieve secrets (like AWS credentials) from **Vault** in a **Jenkins pipeline** and pass them to **Terraform**.

---

##  Jenkins Integration Options

###  1. **Use Jenkins + Vault CLI**

Jenkins pulls secrets from Vault using the CLI before running Terraform.

###  2. **Use Vault Jenkins Plugin** *(Recommended for tighter integration)*

Provides token-based, AppRole-based access.

---

##  Recommended Setup: Using Vault CLI with Jenkins

###  Prerequisites:

* Vault CLI installed on Jenkins agent
* Jenkins credentials configured for Vault token or AppRole
* Vault secrets pre-stored (`vault kv put secret/awscreds ...`)

---

##  Jenkins Pipeline Sample (Declarative)

```groovy
pipeline {
  agent any

  environment {
    VAULT_ADDR = 'http://vault.local:8200'
    VAULT_TOKEN = credentials('vault-token') // Jenkins credentials ID
  }

  stages {
    stage('Get Secrets from Vault') {
      steps {
        sh '''
          export AWS_ACCESS_KEY_ID=$(vault kv get -field=access_key secret/awscreds)
          export AWS_SECRET_ACCESS_KEY=$(vault kv get -field=secret_key secret/awscreds)
          export TF_VAR_access_key=$AWS_ACCESS_KEY_ID
          export TF_VAR_secret_key=$AWS_SECRET_ACCESS_KEY
        '''
      }
    }

    stage('Terraform Init & Apply') {
      steps {
        sh '''
          cd terraform/
          terraform init
          terraform apply -auto-approve
        '''
      }
    }
  }
}
```

---

##  Your Terraform `main.tf` should look like:

```hcl
variable "access_key" {}
variable "secret_key" {}

provider "aws" {
  region     = "us-east-1"
  access_key = var.access_key
  secret_key = var.secret_key
}
```

---

##  Alternative: Use Vault AppRole in Jenkins

More secure than using a static token.

### Steps:

1. Create AppRole in Vault:

```bash
vault auth enable approle

vault write auth/approle/role/jenkins-role \
  secret_id_ttl=60m \
  token_ttl=20m \
  token_max_ttl=1h \
  policies=terraform-policy
```

2. Retrieve Role ID and Secret ID:

```bash
vault read auth/approle/role/jenkins-role/role-id
vault write -f auth/approle/role/jenkins-role/secret-id
```

3. Store `role_id` and `secret_id` in Jenkins Credentials.

4. Jenkins pipeline retrieves a Vault token using:

```bash
VAULT_TOKEN=$(vault write -field=token auth/approle/login role_id=$ROLE_ID secret_id=$SECRET_ID)
```

---

##  Full CI/CD Flow with Jenkins + Vault + Terraform

```
┌────────────┐
│  Jenkins   │
└────┬───────┘
     │ Vault token / AppRole login
     ▼
┌────────────┐
│   Vault    │ ← stores AWS creds / secrets
└────┬───────┘
     │ secrets injected via env vars or tfvars
     ▼
┌────────────┐
│ Terraform  │ ← uses secrets securely
└────────────┘
```

---

##  Benefits of Using Vault in Jenkins + Terraform

* No secrets hardcoded in Terraform or Jenkins.
* Centralized secret management (rotation, TTL, revocation).
* Vault’s access logs provide full audit trails.
* Secrets are short-lived (if dynamic), minimizing exposure.

---





---

##  Dev Mode Limitations

| Limitation                        | Description                                                                           |
| --------------------------------- | ------------------------------------------------------------------------------------- |
|  **Non-persistent**               | All data is stored **in memory** — lost when Vault restarts.                          |
|  **Auto-unsealed**                | Vault is automatically initialized and unsealed — not suitable for production.        |
|  **Root token always exposed**    | A root token is displayed in the terminal — insecure for multi-user/team use.         |
|  **Only 1 unseal key**            | No Shamir’s key split — only for testing.                                             |
|  **TLS is disabled by default**   | All traffic is **HTTP**, not HTTPS — insecure for real Jenkins/Terraform connections. |

---

##  Example: Using Vault Dev Mode in Jenkins + Terraform

### Step 1: Start Vault in Dev Mode

```bash
vault server -dev
```

You’ll see a root token like:

```
Root Token: s.abc123def456
```

Copy this token and use it in your Jenkins credentials or as `VAULT_TOKEN`.

---

### Step 2: Store Secrets in Dev Vault

```bash
vault kv put secret/awscreds access_key=xxx secret_key=yyy
```

---

### Step 3: Jenkins Pipeline (Same as Before)

Use the root token in Jenkins like:

```groovy
environment {
  VAULT_ADDR = 'http://127.0.0.1:8200'
  VAULT_TOKEN = credentials('vault-root-token')
}
```

And run the same Terraform pipeline steps.

---

### ⚠ Pro Tips for Dev Mode

| Tip                                         | Reason                                               |
| ------------------------------------------- | ---------------------------------------------------- |
|  Use for local testing, learning, or demos  | Quick and easy, no config needed                     |
|  Don’t use for real secrets or teams        | Not secure or persistent                             |
|  Restarting Vault?                          | Re-store secrets again manually — they don’t persist |

---



