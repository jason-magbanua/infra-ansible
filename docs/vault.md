# HashiCorp Vault

Native binary install on a dedicated LXC container. Raft backend, single node. TLS terminated upstream by Traefik.

---

## Host

| | |
|---|---|
| **Host** | `hashicorp-vault` |
| **IP** | `10.10.200.40` |
| **Ports** | `8200` (API/UI), `8201` (cluster) |
| **Data dir** | `/var/lib/vault` |
| **Config** | `/etc/vault.d/vault.hcl` |
| **Binary** | `/usr/local/bin/vault` |
| **Service** | `vault.service` (systemd) |
| **UI** | `https://vault.magbanua.xyz` (via Traefik) |

---

## First-time deploy

### Step 1 — Provision the LXC

```bash
cd terraform/
terraform apply
```

### Step 2 — Install Vault binary

```bash
cd ansible/
ansible-playbook playbooks/LXC/setup-vault.yml --ask-vault-pass
```

### Step 3 — Initialize Vault (manual, one-time)

```bash
ssh root@10.10.200.40
export VAULT_ADDR=http://10.10.200.40:8200
vault operator init -key-shares=5 -key-threshold=3
```

The output contains 5 unseal keys and a root token. **Save all of them in Vaultwarden immediately before proceeding.**

### Step 4 — Unseal Vault

```bash
vault operator unseal <Unseal Key 1>
vault operator unseal <Unseal Key 2>
vault operator unseal <Unseal Key 3>

vault status   # Sealed: false confirms success
```

Exit the SSH session.

### Step 5 — Populate secrets file

```bash
cp secrets/vault-vault.yml.example secrets/vault-vault.yml
```

Edit `secrets/vault-vault.yml` and fill in the real values:

```yaml
vault_root_token: "hvs.<from init output>"
vault_db_user: "vault"            # PostgreSQL user Vault will use
vault_db_pass: "<choose a password>"
vault_authentik_secret_key: "<50-char random string>"
```

Then encrypt it:

```bash
ansible-vault encrypt secrets/vault-vault.yml
```

### Step 6 — Configure Vault (engines, roles, AppRole)

```bash
cd ansible/
ansible-playbook playbooks/LXC/setup-vault-config.yml --ask-vault-pass
```

This playbook:
1. Creates the `vault` PostgreSQL user with `CREATEROLE` on `10.10.200.30`
2. Adds a `pg_hba.conf` entry allowing `10.10.200.40` to connect
3. Enables KV v2 at `secret/` and the database engine at `database/`
4. Configures the PostgreSQL connection and creates the `authentik-role` dynamic role
5. Writes static secrets to `secret/authentik/`
6. Enables AppRole auth, creates `ansible-policy` and `ansible-role`
7. Prints the `ansible-role` **role_id** — save it in Vaultwarden

### Step 7 — Revoke root token

Verify AppRole works first:

```bash
ssh root@10.10.200.40
export VAULT_ADDR=http://10.10.200.40:8200

SECRET_ID=$(vault write -f -field=secret_id auth/approle/role/ansible-role/secret-id)
vault write auth/approle/login role_id="<role_id>" secret_id="$SECRET_ID"
# Should return a token with ansible-policy
```

Then revoke the root token:

```bash
export VAULT_TOKEN=<root_token>
vault token revoke "$VAULT_TOKEN"
```

Finally, remove `vault_root_token` from `secrets/vault-vault.yml` and re-encrypt:

```bash
ansible-vault decrypt secrets/vault-vault.yml
# Remove the vault_root_token line
ansible-vault encrypt secrets/vault-vault.yml
```

---

## Unseal after restart

Vault seals itself on every restart. Always manual — never automated.

```bash
ssh root@10.10.200.40
export VAULT_ADDR=http://10.10.200.40:8200
vault operator unseal <Unseal Key 1>
vault operator unseal <Unseal Key 2>
vault operator unseal <Unseal Key 3>
vault status
```

---

## Secrets engines

| Engine | Path | Type |
|--------|------|------|
| KV v2 | `secret/` | Static secrets |
| Database | `database/` | Dynamic credentials |

### KV paths

| Path | Contents |
|------|----------|
| `secret/authentik` | `SECRET_KEY` |

### Dynamic roles

| Role | Engine | DB | Default TTL | Max TTL |
|------|--------|----|-------------|---------|
| `authentik-role` | database | postgresql | 1h | 24h |

Generate a credential:

```bash
vault read database/creds/authentik-role
```

---

## AppRole auth (for Ansible)

`ansible-role` has read-only access to `secret/*` and `database/creds/*`.

### Get role_id (static, safe to store)
```bash
vault read auth/approle/role/ansible-role/role-id
```

### Generate secret_id (generate at runtime, not stored)
```bash
vault write -f auth/approle/role/ansible-role/secret-id
```

### Login
```bash
vault write auth/approle/login role_id="<role_id>" secret_id="<secret_id>"
```

### Policy (ansible-policy)
```hcl
path "secret/*" {
  capabilities = ["read", "list"]
}

path "database/creds/*" {
  capabilities = ["read"]
}
```

---

## Traefik routing

The `vault` role deploys `/etc/traefik/conf.d/vault.yml` to the Traefik LXC via `delegate_to: traefik`. Hot-reloaded automatically.

Route: `https://vault.magbanua.xyz` → `http://10.10.200.40:8200`

---

## vault.hcl

```hcl
storage "raft" {
  path    = "/var/lib/vault"
  node_id = "vault-01"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = true
}

api_addr      = "http://10.10.200.40:8200"
cluster_addr  = "http://10.10.200.40:8201"
disable_mlock = true   # required for unprivileged LXC containers
ui            = true
```

---

## Secrets file reference

`secrets/vault-vault.yml` (Ansible Vault encrypted). See `secrets/vault-vault.yml.example` for the full template.

| Variable | Used by | Notes |
|---|---|---|
| `vault_root_token` | `vault_config` role | Remove after revoking root token |
| `vault_db_user` | `vault_config` role | PostgreSQL user for DB secrets engine |
| `vault_db_pass` | `vault_config` role | |
| `vault_authentik_secret_key` | `vault_config` role | Written to `secret/authentik/SECRET_KEY` |
