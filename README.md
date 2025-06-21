

---
<div align="center">
  
#  **HashiCorp Vault**

</div>

## Why Use Vault?

In a world where digital assets are more valuable than ever, protecting sensitive information is non-negotiable. Whether it's API keys, credentials, or configuration values, secrets can easily become vulnerabilities if mishandled. **HashiCorp Vault** provides a centralized, robust, and scalable solution to manage these secrets securely — eliminating the risks of hardcoded credentials and scattered secrets.

---

##  Getting Started: Installing Vault

Before we dive into securing secrets, let’s set up Vault on your machine.

### 1. Download Vault

Visit the [official Vault download page](https://www.vaultproject.io/downloads) and grab the latest version for your OS.

### 2. Install Vault

Install Vault using the method appropriate for your system. For macOS, for example:

```bash
brew install vault
```

### 3. Start Vault in Development Mode

Launch the Vault server with:

```bash
vault server -dev
```

This runs Vault in development mode, great for experimentation — but not for production.

### 4. Verify Vault is Running

Open a new terminal window and run:

```bash
vault status
```

You’ll get status output confirming initialization, seal status, and shard configuration.

---

##  Configuring Vault

Once Vault is live, we can begin setting up core components like authentication, policies, and secret engines.

### 1. Authentication Methods

Vault supports multiple auth mechanisms: tokens, GitHub, LDAP, Kubernetes, etc. To keep it simple:

```bash
vault auth enable token
```

### 2. Define Access Policies

Policies define what actions a user or app can take. Create a basic read-only policy:

```hcl
# readonly.hcl
path "secret/*" {
  capabilities = ["read"]
}
```

Apply it:

```bash
vault policy write readonly readonly.hcl
```

### 3. Enable Secret Engines

Secret engines manage different types of secrets. The Key/Value engine is the most basic:

```bash
vault secrets enable -path=secret kv
```

### 4. Store and Retrieve Secrets

Save a secret:

```bash
vault kv put secret/mysecret myvalue=s3cr3t
```

Read it:

```bash
vault kv get secret/mysecret
```

---

##  Secrets Management: Core Operations

Now you're ready to start handling real secrets.

### Storing Secrets

```bash
vault kv put secret/myapp password=hunter2
```

### Reading Secrets

```bash
vault kv get secret/myapp
```

### Updating Secrets

```bash
vault kv put secret/myapp password=newpassword
```

### Deleting Secrets

```bash
vault kv delete secret/myapp
```

---

##  Dynamic Secrets: Just-in-Time Credentials

Unlike static secrets, dynamic secrets are generated on-demand and expire automatically.

### Example: MySQL Dynamic Credentials

#### 1. Enable the Engine

```bash
vault secrets enable database
```

#### 2. Configure DB Connection

```bash
vault write database/config/mydb \
  plugin_name=mysql-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(127.0.0.1:3306)/" \
  allowed_roles="readonly" \
  username="root" \
  password="rootpassword"
```

#### 3. Create Role

```bash
vault write database/roles/readonly \
  db_name=mydb \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; GRANT SELECT ON *.* TO '{{name}}'@'%';" \
  default_ttl="1h" \
  max_ttl="24h"
```

#### 4. Generate Credentials

```bash
vault read database/creds/readonly
```

---

##  Fine-Grained Access Control

Use policies to enforce strict access control.

### Restricting Access to One Secret

#### 1. Define Policy

```hcl
# restricted.hcl
path "secret/myapp" {
  capabilities = ["create", "read", "update", "patch", "delete", "list"]
}
```

#### 2. Apply and Assign to User

```bash
vault policy write restricted restricted.hcl
vault write auth/userpass/users/jane password=jane123 policies=restricted
```

---

##  Real-World Use Case: API Key Management

Store and securely access an external API key in your app.

```bash
vault kv put secret/api_key value=myapikey123
```

Define policy:

```hcl
# api_access.hcl
path "secret/api_key" {
  capabilities = ["read"]
}
```

Apply and assign:

```bash
vault policy write api_access api_access.hcl
vault write auth/userpass/users/appuser password=apppassword policies=api_access
```

In your app:

```bash
vault kv get -field=value secret/api_key
```

---

##  Advanced Vault Usage

### Dynamic AWS Credentials

```bash
vault secrets enable aws
vault write aws/config/root \
  access_key=<your-aws-access-key> \
  secret_key=<your-aws-secret-key> \
  region=us-east-1

vault write aws/roles/my-role \
  credential_type=iam_user \
  policy_arns=arn:aws:iam::aws:policy/ReadOnlyAccess

vault read aws/creds/my-role
```

### Lease and Revocation

```bash
vault write aws/config/lease lease=1h lease_max=24h
vault read sys/leases/lookup/aws/creds/my-role/<lease-id>
vault lease revoke aws/creds/my-role/<lease-id>
```

### Transit Engine: Encryption as a Service

```bash
vault secrets enable transit
vault write -f transit/keys/my-key
vault write transit/encrypt/my-key plaintext=$(echo "my secret data" | base64)
vault write transit/decrypt/my-key ciphertext=<ciphertext>
```

---

##  Deployment Scenario: Secure Web App

### 1. Dynamic Database Access

```bash
vault secrets enable database

vault write database/config/mydb \
  plugin_name=mysql-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(127.0.0.1:3306)/" \
  allowed_roles="app-role" \
  username="root" \
  password="rootpassword"

vault write database/roles/app-role \
  db_name=mydb \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; GRANT SELECT ON *.* TO '{{name}}'@'%';" \
  default_ttl="1h" \
  max_ttl="24h"
```

### 2. Store and Access API Key

```bash
vault kv put secret/api_key value=myapikey123

echo '
path "secret/api_key" {
  capabilities = ["read"]
}
' | vault policy write app-policy -

vault write auth/userpass/users/appuser password=apppassword policies=app-policy
```

### 3. Application Integration

```bash
VAULT_TOKEN=$(vault login -method=userpass username=appuser password=apppassword -format=json | jq -r ".auth.client_token")
API_KEY=$(vault kv get -field=value secret/api_key)
DB_CREDS=$(vault read database/creds/app-role -format=json)
DB_USER=$(echo $DB_CREDS | jq -r ".data.username")
DB_PASS=$(echo $DB_CREDS | jq -r ".data.password")
```

---

## 🛡 Security Best Practices

### 1. Enable Audit Logs

```bash
vault audit enable file file_path=/var/log/vault_audit.log
```

### 2. Use TLS

```hcl
listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = 0
  tls_cert_file = "/path/to/cert.pem"
  tls_key_file  = "/path/to/key.pem"
}
```

### 3. Restrict Access via Firewall

```bash
sudo iptables -A INPUT -p tcp -s 192.168.1.100 --dport 8200 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 8200 -j DROP
```

### 4. Use Strong Authentication (e.g., LDAP)

```bash
vault auth enable ldap
vault write auth/ldap/config \
  url="ldap://ldap.example.com" \
  binddn="cn=admin,dc=example,dc=com" \
  bindpass="adminpassword" \
  userdn="ou=users,dc=example,dc=com"
```

### 5. Enforce Least Privilege

```hcl
# minimal_policy.hcl
path "secret/data/*" {
  capabilities = ["read"]
}
```

```bash
vault policy write minimal minimal_policy.hcl
```

### 6. Automate Rotation

Vault's leasing features ensure secrets expire and regenerate without manual intervention.

---

##  Backup & Restore

```bash
vault operator raft snapshot save /path/to/backup/snapshot
vault operator raft snapshot restore /path/to/backup/snapshot
```

---

##  Monitoring

Use Prometheus with Vault's telemetry endpoint:

```hcl
telemetry {
  prometheus_retention_time = "24h"
  prometheus_retention_bytes = "10MB"
}
```

---

##  Production Example Config (`config.hcl`)

```hcl
listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 0
  tls_cert_file = "/etc/vault/tls/cert.pem"
  tls_key_file  = "/etc/vault/tls/key.pem"
}

storage "raft" {
  path    = "/var/lib/vault"
  node_id = "vault-node-1"
}

api_addr = "https://vault.example.com:8200"
cluster_addr = "https://vault.example.com:8201"
ui = true
```

---

##  Final Thoughts

Vault is more than just a place to stash secrets — it's a full-fledged security platform that empowers teams to manage sensitive information in a modern, automated, and secure way. By following best practices, implementing dynamic access, and integrating with your systems, you ensure your secrets stay safe — and stay secret.
