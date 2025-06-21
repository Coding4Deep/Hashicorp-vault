
---

##  Why Use Vault with Terraform?

* Securely inject secrets into Terraform without hardcoding them.
* Store secrets centrally in Vault, manage TTLs, and auto-rotate credentials.
* Enable least privilege and audit access.

---

##  Use Case: Inject AWS credentials (or any secret) from Vault into Terraform

###  Step 1: Store AWS Credentials in Vault

```bash
vault kv put secret/awscreds access_key=AKIAxxxxxxxx secret_key=xxxxxxxxxxxxx
```

This stores your secrets in Vault's KV engine under the path `secret/awscreds`.

---

###  Step 2: Enable Vault KV Secrets Engine (if not already)

```bash
vault secrets enable -path=secret kv
```

---

###  Step 3: Configure Vault Provider in Terraform

```hcl
provider "vault" {
  address = "http://127.0.0.1:8200"
  token   = var.vault_token  # Don't hardcode, pass via environment or tfvars
}
```

You can pass the token via:

```bash
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=s.XXXXXX
```

---

###  Step 4: Read Secrets from Vault using `vault_generic_secret` data source

```hcl
data "vault_generic_secret" "awscreds" {
  path = "secret/awscreds"
}
```

Now, access the secrets like:

```hcl
output "access_key" {
  value = data.vault_generic_secret.awscreds.data["access_key"]
}

output "secret_key" {
  value = data.vault_generic_secret.awscreds.data["secret_key"]
}
```

---

###  Step 5: Use in AWS Provider

```hcl
provider "aws" {
  region     = "us-east-1"
  access_key = data.vault_generic_secret.awscreds.data["access_key"]
  secret_key = data.vault_generic_secret.awscreds.data["secret_key"]
}
```

---

##  Example: Full `main.tf` Sample

```hcl
provider "vault" {
  address = "http://127.0.0.1:8200"
  token   = var.vault_token
}

data "vault_generic_secret" "awscreds" {
  path = "secret/awscreds"
}

provider "aws" {
  region     = "us-east-1"
  access_key = data.vault_generic_secret.awscreds.data["access_key"]
  secret_key = data.vault_generic_secret.awscreds.data["secret_key"]
}

resource "aws_s3_bucket" "example" {
  bucket = "my-secure-bucket-terraform"
  acl    = "private"
}
```

---

##  Alternative: Dynamic Secrets via Vault AWS Secrets Engine

If you're using **Vault to generate dynamic AWS credentials**, replace the KV engine with:

```hcl
data "vault_aws_access_credentials" "creds" {
  backend = "aws"          # Secrets engine path
  role    = "readonly"     # Role created in Vault
}
```

And use like:

```hcl
provider "aws" {
  access_key = data.vault_aws_access_credentials.creds.access_key
  secret_key = data.vault_aws_access_credentials.creds.secret_key
  region     = "us-east-1"
}
```

---

##  Terraform Variable Setup (optional but recommended)

In `variables.tf`:

```hcl
variable "vault_token" {
  description = "Vault token for auth"
  type        = string
  sensitive   = true
}
```

In `terraform.tfvars` or environment variables:

```hcl
vault_token = "s.XXXXXXX"
```

---

##  Security Best Practices

* Use dynamic secrets where possible.
* Use short TTL and auto-revocation features of Vault.
* Never hardcode secrets or tokens in `.tf` files — use environment variables or encrypted `.tfvars`.
* Use Vault's policies to restrict access to only the secrets your Terraform module needs.

---


