# Loki & Grafana Alloy

Log aggregation stack. Loki stores logs; Alloy collects and forwards them.

---

## Loki

**Role:** `roles/loki/`
**Playbook:** `playbooks/LXC/setup-loki.yml`
**Host:** `loki` inventory group — LXC at `10.10.200.23`

### What the role does

1. Creates `loki` system user (nologin)
2. Downloads Loki binary from GitHub releases (`loki-linux-amd64.zip`) and installs to `/usr/local/bin/loki`
3. Templates `/etc/loki/loki-config.yml` (single-binary, filesystem storage)
4. Deploys `/etc/systemd/system/loki.service` with systemd hardening
5. Enables and starts the `loki` service

### Binary install pattern

Same as Prometheus — download from GitHub releases, extract, copy binary, idempotent via `stat` check + `register`. Loki releases use `.zip` (not `.tar.gz`); the archive contains a single binary named `loki-linux-amd64`.

### Configuration

Single-binary mode with local filesystem storage:

| Setting               | Default               |
|-----------------------|-----------------------|
| `loki_version`        | `3.4.2`               |
| `loki_port`           | `3100`                |
| `loki_grpc_port`      | `9096`                |
| `loki_storage_path`   | `/var/lib/loki`       |
| `loki_retention_period` | `30d`              |
| Schema                | `v13` / `tsdb`        |

### Running the playbook

```bash
cd ansible/
ansible-playbook playbooks/LXC/setup-loki.yml
```

### Verify

```bash
curl -s http://10.10.200.23:3100/ready
# should return: ready
```

### Traefik integration

`roles/traefik/templates/conf.d/loki.yml.j2` creates a `loki.{{ traefik_domain }}` router backed by `http://10.10.200.23:3100`. Re-run the Traefik playbook to activate:

```bash
ansible-playbook playbooks/LXC/setup-traefik.yml --ask-vault-pass
```

### Grafana datasource

The Loki datasource is now active (uncommented) in `roles/grafana/templates/provisioning/datasources.yml.j2` with `uid: loki`. Re-run the Grafana playbook to provision it:

```bash
ansible-playbook playbooks/LXC/setup-grafana.yml --ask-vault-pass
```

---

## Grafana Alloy

**Role:** `roles/alloy/`
**Playbook:** `playbooks/LXC/setup-alloy.yml`
**Host:** `alloy` inventory group — LXC at `10.10.200.24`

### What the role does

1. Adds Grafana APT repo + GPG key (same repo as Grafana)
2. Installs the `alloy` package
3. Templates `/etc/alloy/config.alloy` (River syntax)
4. Templates `/etc/default/alloy` to set the HTTP listen address
5. Enables and starts the `alloy` service

### APT install note

Alloy uses the same `https://apt.grafana.com` repository as Grafana. The GPG key is the same (`/etc/apt/keyrings/grafana.gpg`). If the Grafana role has already run on the same host, the key/repo steps are idempotent.

### Configuration

| Setting                 | Default         |
|-------------------------|-----------------|
| `alloy_version`         | `latest`        |
| `alloy_http_port`       | `12345`         |
| `alloy_loki_push_port`  | `3500`          |
| `loki_ip`               | `10.10.200.23`  |
| `loki_port`             | `3100`          |

### What Alloy collects

- **Local journal logs** — `loki.source.journal` reads the systemd journal on the Alloy LXC and relabels `host`, `unit`, and `level` labels before forwarding to Loki.
- **Remote log push endpoint** — `loki.source.api` listens on `0.0.0.0:3500` (Loki-compatible HTTP push). Other hosts can ship logs to Alloy's push endpoint rather than directly to Loki.

All log streams are forwarded to `loki.write.default` → `http://10.10.200.23:3100/loki/api/v1/push`.

### Running the playbook

```bash
cd ansible/
ansible-playbook playbooks/LXC/setup-alloy.yml
```

### Verify

```bash
# Alloy UI and health
curl -s http://10.10.200.24:12345/-/ready
# should return: Alloy is ready.

# Confirm logs arriving in Loki
curl -s 'http://10.10.200.23:3100/loki/api/v1/query?query={job="systemd-journal"}&limit=1'
```

### Shipping logs from other hosts to Alloy

Point any Promtail/Loki agent or Alloy instance on other hosts at the push endpoint:

```
http://10.10.200.24:3500/loki/api/v1/push
```

---

## Key design decisions

- **Alloy as a forwarding proxy** — rather than having every host push directly to Loki, they push to Alloy's `loki.source.api` endpoint. This decouples log producers from the Loki backend address.
- **No Alloy Traefik route** — the Alloy UI (port 12345) is internal-only; no external exposure needed for a homelab.
- **Retention at Loki layer** — `limits_config.retention_period: 30d` controls storage growth without requiring external compactor setup.
