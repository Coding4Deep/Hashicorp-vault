
---

##  Vault Core CLI Commands

###  Initialization & Unseal

```bash
vault operator init                            # Initialize Vault
vault operator unseal                          # Manually unseal Vault using unseal key
vault operator seal                            # Seal the Vault
vault status                                   # Check Vault status
vault operator rekey                           # Rotate the unseal keys
vault operator rekey -init                     # Start rekey operation
vault operator rekey -cancel                   # Cancel ongoing rekey operation
vault operator rotate                          # Rotate encryption key used for storage
vault operator raft snapshot save <file>       # Create snapshot of Vault data
vault operator raft snapshot restore <file>    # Restore snapshot
```

---

##  Authentication & Tokens

###  Login / Logout

```bash
vault login                                    # Login to Vault
vault login -method=userpass username=xyz     # Login with userpass
vault logout                                   # Logout
```

###  Token Management

```bash
vault token create                             # Create a new token
vault token lookup                             # View current token info
vault token revoke                             # Revoke token
vault token renew                              # Renew token
vault token capabilities <path>                # Check token permissions for path
```

###  Enable Authentication Backends

```bash
vault auth list                                # List enabled auth methods
vault auth enable <method>                     # Enable auth method (e.g., userpass, ldap, github)
vault auth disable <path>                      # Disable auth method
vault write auth/userpass/users/<user> ...     # Create userpass user
vault read auth/userpass/users/<user>          # View user info
vault delete auth/userpass/users/<user>        # Delete user
```

---

##  Secrets Management

###  Key/Value (KV) Secrets Engine

```bash
vault secrets list                             # List enabled secret engines
vault secrets enable -path=secret kv           # Enable kv secrets engine
vault secrets disable secret                   # Disable secret engine
vault kv put secret/<path> key=value           # Store a secret
vault kv get secret/<path>                     # Retrieve secret
vault kv delete secret/<path>                  # Delete latest version
vault kv destroy secret/<path> -versions=N     # Permanently delete specific versions
vault kv metadata delete secret/<path>         # Delete all versions and metadata
vault kv list secret/                          # List secrets under a path
```

---

###  Dynamic Secrets (Database / AWS / etc.)

```bash
vault secrets enable database                  # Enable database secrets engine
vault write database/config/<db-name> ...      # Configure DB connection
vault write database/roles/<role> ...          # Create DB role
vault read database/creds/<role>               # Generate dynamic credentials

vault secrets enable aws                       # Enable AWS secrets engine
vault write aws/config/root ...                # Configure AWS
vault write aws/roles/<role> ...               # Create IAM role
vault read aws/creds/<role>                    # Generate dynamic AWS credentials
```

---

###  Transit Secrets Engine

```bash
vault secrets enable transit                   # Enable transit
vault write -f transit/keys/<key>              # Create encryption key
vault write transit/encrypt/<key> plaintext=<base64>
vault write transit/decrypt/<key> ciphertext=<ciphertext>
vault delete transit/keys/<key>                # Delete key
```

---

##  Policy & Permissions

```bash
vault policy list                              # List policies
vault policy write <name> <file.hcl>           # Create/update policy
vault policy read <name>                       # Read policy
vault policy delete <name>                     # Delete policy
```

---

##  Leases and Renewals

```bash
vault lease revoke <lease_id>                  # Revoke specific lease
vault lease revoke -prefix <prefix>            # Revoke all leases under a path
vault lease lookup <lease_id>                  # View lease details
vault renew <lease_id>                         # Renew a lease
```

---

##  Audit Logging

```bash
vault audit list                               # List enabled audit devices
vault audit enable file file_path=/var/log/vault_audit.log
vault audit disable <path>                     # Disable audit device
```

---

##  Debugging & Tools

```bash
vault read <path>                              # Read any Vault path
vault write <path> key=value                   # Write data to path
vault delete <path>                            # Delete data from a path
vault list <path>                              # List keys in a path
vault ssh <path>                               # SSH with Vault
vault plugin list                              # List plugins
vault plugin register ...                      # Register a plugin
```

---

##  Telemetry & Monitoring

```bash
vault monitor                                  # Real-time logs from the Vault server
vault debug                                    # Gather detailed Vault debug info
```

---

##  Miscellaneous Useful Commands

```bash
vault version                                  # Vault CLI version
vault -h / vault --help                        # Help menu
vault plugin catalog list secrets              # List available secret plugins
vault server -config=config.hcl                # Start server with config
```

---

##  Example Vault CLI Usage Flow

```bash
# Enable KV
vault secrets enable -path=secret kv

# Add secret
vault kv put secret/myapp password=hunter2

# Read secret
vault kv get secret/myapp

# Create policy
vault policy write mypolicy ./mypolicy.hcl

# Enable userpass
vault auth enable userpass

# Add user
vault write auth/userpass/users/jane password=janepass policies=mypolicy

# Login as user
vault login -method=userpass username=jane password=janepass
```

---

##  `vault operator init` — **Initialization of Real Vault Instance**

###  Purpose:

This command is used to **initialize a real Vault deployment** (non-dev mode). It sets up the master key, generates unseal keys, and prepares Vault for production use.

###  What it does:

* Generates a **master key**.
* Splits the master key into **unseal key shards** using **Shamir’s Secret Sharing**.
* Creates the initial **root token**.
* Stores an **encrypted DEK (data encryption key)** in the storage backend (like Consul, S3, etc).
* Prepares Vault to be unsealed using `vault operator unseal`.

###  Usage:

```bash
vault operator init -key-shares=5 -key-threshold=3
```

###  When to use:

* When you run Vault in **server mode** (`vault server -config=config.hcl`).
* **Only run once** per Vault deployment. Initialization is a **one-time operation**.

---

## 🔸 `vault server -dev` — **Development/Test Mode**

###  Purpose:

Starts Vault in **development mode** for quick testing and learning. It’s not secure and should **never be used in production**.

###  What it does:

* **Skips `vault operator init`** — Vault auto-initializes itself.
* Uses a **single in-memory unseal key** and **auto-unseals** itself.
* Generates a **root token automatically**.
* Stores everything **in-memory** — all data is wiped on restart.
* Default settings are insecure (e.g., no TLS, root access exposed).

###  Usage:

```bash
vault server -dev
```

###  When to use:

* For **local development**, **learning**, or **trying out commands**.
* When you **don’t want to deal with unseal/init/config complexity**.

---



