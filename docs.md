#  HashiCorp Vault: Concepts, Terms & Techniques Explained

This documentation provides an in-depth explanation of every major term, concept, and mechanism used in HashiCorp Vault. It serves as a knowledge base for beginners and professionals who want to understand Vault thoroughly.

---

##  1. What is Vault?

Vault is a secrets management tool developed by HashiCorp that allows secure storage, access control, and dynamic generation of sensitive information such as passwords, API keys, and database credentials.

---

##  2. Core Vault Concepts

###  Initialization (`vault operator init`)

* The first step after starting a Vault server.
* Initializes the encryption key (DEK), creates the unseal keys, and generates a root token.

###  Sealing & Unsealing

* **Sealed Vault:** Encrypted barrier is locked. Data is inaccessible.
* **Unseal:** Requires unseal keys (from Shamir's secret sharing) to decrypt the DEK and access secrets.
* **vault operator unseal:** Used to input each unseal key.

###  Shamir's Secret Sharing

* A cryptographic technique that splits a key into multiple parts.
* A threshold number of keys are needed to reconstruct the original master key.

###  Root Token

* A highly privileged access token generated during initialization.
* Used to create other tokens and configure Vault.

###  Barrier

* A security layer that encrypts/decrypts all data stored in the backend.
* Vault is inaccessible until the barrier is decrypted (i.e., Vault is unsealed).

###  Auto-Unseal

* Automatically unseals Vault using a cloud KMS like AWS KMS, Azure Key Vault, or GCP KMS.
* Ideal for production to avoid manual unseal on restart.

---

##  3. Storage Backends

* Used to store encrypted secrets, metadata, leases, etc.
* Examples: Consul, S3, DynamoDB, Raft.
* Vault encrypts all data before storing it in the backend.

---

##  4. Authentication Methods

Vault supports multiple ways to authenticate users and machines:

* `userpass`: Username/password.
* `github`: GitHub OAuth.
* `ldap`: Enterprise directory.
* `kubernetes`: Service account tokens.
* `token`: Static or dynamic tokens.
* `approle`: Role ID and secret ID for machine-to-machine auth.

---

##  5. Access Control (Policies)

###  Policies

* Define what actions users or apps can perform.
* Written in HCL (HashiCorp Configuration Language).

###  Capabilities

Examples:

* `create`, `read`, `update`, `delete`
* `list`, `sudo`

Example policy:

```hcl
path "secret/*" {
  capabilities = ["read"]
}
```

---

##  6. Secret Engines

Secret engines are components that manage different types of secrets:

###  Key/Value (KV)

* Stores static secrets as key-value pairs.

###  Database Secrets Engine

* Generates dynamic, time-bound credentials for supported databases.

### ☁ AWS Secrets Engine

* Creates temporary AWS IAM credentials on the fly.

###  Transit Secrets Engine

* Encryption as a service; does not store secrets, only encrypts/decrypts data.

###  PKI Secrets Engine

* Issues dynamic TLS/SSL certificates.

---

##  7. Leases and TTL

* Every secret has a **lease** — a TTL (time-to-live).
* Vault automatically revokes secrets when TTL expires.

### Lease Operations

* `vault lease revoke <id>`: Revoke specific lease.
* `vault lease renew <id>`: Extend a lease.

---

##  8. Auditing

* Vault supports enabling audit logs to track all access and operations.

```bash
vault audit enable file file_path=/var/log/vault_audit.log
```

---

##  9. Configuration Files

Vault server is usually started with a configuration file (e.g., `config.hcl`).

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
```

---

##  10. Monitoring & Telemetry

Vault emits metrics which can be collected by Prometheus:

```hcl
telemetry {
  prometheus_retention_time = "24h"
  prometheus_retention_bytes = "10MB"
}
```

---

##  11. Best Practices

* Enable TLS for all communications.
* Use audit logs.
* Use short TTLs for secrets.
* Enforce least privilege with policies.
* Periodically rotate root tokens and unseal keys.
* Enable auto-unseal for production.
* Backup Raft snapshots regularly.

---

##  Conclusion

HashiCorp Vault offers powerful tools for secrets management, encryption, and access control. Understanding its terminology and internal mechanisms helps deploy secure, production-grade infrastructure.


