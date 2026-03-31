# Authentik SSO

Authentik runs as a Docker Compose stack on `docker-host` (10.10.200.50), bound to a dedicated secondary IP `10.10.200.42`. It uses external PostgreSQL (10.10.200.30) and Redis (10.10.200.31) LXCs shared across services.

---

## Infrastructure

PostgreSQL and Redis LXCs are already defined in `terraform/main.tf` — no Terraform changes needed. The `10.10.200.42` IP alias is provisioned by `playbooks/VMs/setup-docker-host.yml` via a systemd oneshot service.

---

## PostgreSQL role

**Role:** `roles/postgresql/`
**Playbook:** `playbooks/LXC/setup-postgresql.yml`
**Host:** `postgresql` inventory group — LXC at `10.10.200.30`

### What the role does

1. Installs `postgresql-16` and `python3-psycopg2` (required by `community.postgresql` modules)
2. Sets `listen_addresses` in `postgresql.conf` to bind on `127.0.0.1` and `10.10.200.30`
3. Adds an `pg_hba.conf` entry allowing the `authentik` user from `docker-host` (10.10.200.50) using `scram-sha-256`
4. Creates the `authentik` database user (password from Vault)
5. Creates the `authentik` database owned by that user

### Key variables

| Variable | Default |
|----------|---------|
| `postgresql_version` | `16` |
| `postgresql_listen_addresses` | `127.0.0.1, 10.10.200.30` |
| `authentik_db_name` | `authentik` |
| `authentik_db_user` | `authentik` |
| `docker_host_ip` | `10.10.200.50` |
| `authentik_db_password` | **Vault** |

### Running the playbook

```bash
ansible-playbook playbooks/LXC/setup-postgresql.yml --ask-vault-pass
```

### Verify

```bash
# From docker-host, test the connection
psql -h 10.10.200.30 -U authentik -d authentik -c '\conninfo'
```

---

## Redis role

**Role:** `roles/redis/`
**Playbook:** `playbooks/LXC/setup-redis.yml`
**Host:** `redis` inventory group — LXC at `10.10.200.31`

### What the role does

1. Installs `redis-server` via APT (Ubuntu 24.04 ships Redis 7)
2. Updates `/etc/redis/redis.conf` via `lineinfile` (does not replace the full config):
   - `bind 127.0.0.1 10.10.200.31`
   - `protected-mode no` (safe with explicit bind addresses)
   - `maxmemory 256mb`
   - `maxmemory-policy allkeys-lru`

No password is set — Redis is accessible only from within VLAN 200.

### Running the playbook

```bash
ansible-playbook playbooks/LXC/setup-redis.yml
```

### Verify

```bash
redis-cli -h 10.10.200.31 ping
# PONG
```

---

## Authentik role

**Role:** `roles/authentik/`
**Playbook:** `playbooks/LXC/setup-authentik.yml`
**Host:** `docker-host` — stack bound to `10.10.200.42:9000`

### Prerequisites

- `setup-docker-host.yml` must have run to create the `10.10.200.42` IP alias
- PostgreSQL and Redis must be provisioned and reachable
- `secrets/authentik-vault.yml` must exist and be vault-encrypted

### What the role does

1. Creates `/home/infra/docker/authentik/` with `data/media`, `data/certs`, `data/custom-templates` subdirectories
2. Templates `.env` from Vault vars (mode `0600`)
3. Templates `docker-compose.yml` (server + worker services)
4. Pulls `ghcr.io/goauthentik/server:{{ authentik_version }}`
5. Deploys the stack with `community.docker.docker_compose_v2`

### Stack layout

| Service | Image | Role |
|---------|-------|------|
| `server` | `ghcr.io/goauthentik/server` | Web UI + API on port 9000/9443 |
| `worker` | `ghcr.io/goauthentik/server` | Background tasks, certificate management |

The worker runs as `root` and mounts `/var/run/docker.sock` for Authentik outpost container management.

### Key variables

| Variable | Default |
|----------|---------|
| `authentik_version` | `2024.12.3` |
| `authentik_ip` | `10.10.200.42` |
| `authentik_port` | `9000` |
| `authentik_stack_dir` | `/home/infra/docker/authentik` |
| `authentik_secret_key` | **Vault** |
| `authentik_db_password` | **Vault** |
| `authentik_bootstrap_password` | **Vault** |
| `authentik_bootstrap_email` | **Vault** |

### Secrets setup

```bash
cp secrets/authentik-vault.yml.example secrets/authentik-vault.yml
# Edit authentik-vault.yml with real values
# Generate secret_key:
openssl rand -base64 60

ansible-vault encrypt secrets/authentik-vault.yml
```

### Running the playbook

```bash
ansible-playbook playbooks/LXC/setup-authentik.yml --ask-vault-pass
```

### Verify

```bash
curl -s http://10.10.200.42:9000/-/health/live/
# {"status": "ok"}

curl -s https://authentik.magbanua.xyz/-/health/live/
# {"status": "ok"}
```

Then open `https://authentik.magbanua.xyz/if/flow/initial-setup/` to complete the initial admin setup (if `AUTHENTIK_BOOTSTRAP_*` vars are not set).

---

## Traefik integration

`roles/traefik/templates/conf.d/authentik.yml.j2` routes `authentik.{{ traefik_domain }}` → `http://10.10.200.42:9000`. Re-run the Traefik playbook to activate:

```bash
ansible-playbook playbooks/LXC/setup-traefik.yml --ask-vault-pass
```

---

## Full run order

```bash
# 1. Provision secondary IP on docker-host (if not already done)
ansible-playbook playbooks/VMs/setup-docker-host.yml

# 2. PostgreSQL — creates DB and user
ansible-playbook playbooks/LXC/setup-postgresql.yml --ask-vault-pass

# 3. Redis
ansible-playbook playbooks/LXC/setup-redis.yml

# 4. Authentik stack
ansible-playbook playbooks/LXC/setup-authentik.yml --ask-vault-pass

# 5. Traefik route
ansible-playbook playbooks/LXC/setup-traefik.yml --ask-vault-pass
```

---

## Key design decisions

- **External PostgreSQL and Redis** — shared LXCs rather than bundled containers keep the Authentik compose minimal and allow other services to reuse the same DB/cache infrastructure.
- **Secondary IP binding** — Authentik binds to `10.10.200.42` rather than the docker-host primary IP (`10.10.200.50`), giving it a clean dedicated address for Traefik routing without port conflicts.
- **`.env` mode `0600`** — the env file contains the `AUTHENTIK_SECRET_KEY` and DB password; strict permissions prevent other processes from reading it.
- **`community.postgresql` collection** — added to `collections/requirements.yml`; run `ansible-galaxy collection install -r collections/requirements.yml -p collections/` to install.
