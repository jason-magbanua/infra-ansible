# VyOS Router Configuration

Automates the full VyOS router configuration via the `vyos.vyos` Ansible collection. The playbook is idempotent — re-running it applies only what has drifted from the declared state.

---

## Network Overview

```
Internet
   │
  eth0 (DHCP) ──── transit0, Home ISP uplink
   │
 VyOS (172.16.1.182 / 192.168.8.182 / 10.10.200.1)
   │
  wg0 ──────────── 10.255.255.2/29 — WireGuard tunnel to OVH VPS (51.79.161.145)
   │
  eth1 ─────────── 802.1Q trunk to Proxmox host
    ├── vlan100 ── 10.10.100.1/29 — DMZ (public services)
    └── vlan200 ── 10.10.200.1/24 — SERVERS (internal)
```

---

## File Structure

```
playbooks/vyos/
├── vyos.yml          # Main playbook
└── vars/
    └── main.yml      # All non-sensitive variables
```

Secrets live outside the repo:

```
/opt/infra/secrets/
└── vyos-vault.yml    # Sensitive variables (never committed)
```

Connection settings live with the inventory:

```
inventory/lab/group_vars/
└── vyos.yml          # ansible_network_os, ansible_connection, ansible_user
```

---

## Prerequisites

Install the `vyos.vyos` collection:

```bash
ansible-galaxy collection install -r collections/requirements.yml -p collections/
```

Ensure the secrets file exists at `/opt/infra/secrets/vyos-vault.yml` with the required variables (see [Variables](#variables) below).

---

## Usage

**Full configuration run:**

```bash
ansible-playbook playbooks/vyos/vyos.yml
```

**Run only specific sections using tags:**

```bash
ansible-playbook playbooks/vyos/vyos.yml --tags interfaces
ansible-playbook playbooks/vyos/vyos.yml --tags interfaces,routes
```

---

## Tags Reference

| Tag          | What it configures                                           |
|--------------|--------------------------------------------------------------|
| `system`     | Hostname, timezone, commit revisions, console, syslog        |
| `auth`       | Login user, encrypted password, SSH public key               |
| `interfaces` | eth0 (WAN), eth1 (trunk), VLAN 100/200 subinterfaces, wg0   |
| `dhcp`       | DHCP server for the SERVERS subnet (10.10.200.0/24)          |
| `ntp`        | NTP servers and allowed client addresses                     |
| `ssh`        | SSH listen addresses                                         |
| `routes`     | Static routes                                                |

---

## Variables

### `vars/main.yml` (non-sensitive)

Key variables — see the file for the full list.

| Variable                    | Description                                  |
|-----------------------------|----------------------------------------------|
| `vyos_hostname`             | Router hostname (`vyos`)                     |
| `vyos_timezone`             | System timezone (`Asia/Manila`)              |
| `vyos_vlan100_address`      | DMZ gateway IP (`10.10.100.1/29`)            |
| `vyos_vlan200_address`      | SERVERS gateway IP (`10.10.200.1/24`)        |
| `vyos_wg0_address`          | WireGuard tunnel address (`10.255.255.2/29`) |
| `vyos_wg0_peer_endpoint`    | VPS public IP:port (`51.79.161.145:51830`)   |
| `vyos_wg0_keepalive`        | WireGuard keepalive interval (`25`)          |
| `vyos_dhcp_network_name`    | DHCP shared network name (`SERVERS`)         |
| `vyos_dhcp_subnet`          | DHCP subnet (`10.10.200.0/24`)               |
| `vyos_dhcp_range_start/stop`| DHCP range (`.101` – `.200`)                 |
| `vyos_dhcp_lease`           | Lease time in seconds (`86400`)              |
| `vyos_ssh_listen_addresses` | SSH bind addresses                           |
| `vyos_static_routes`        | List of `{prefix, next_hop, description}`    |

### `/opt/infra/secrets/vyos-vault.yml` (sensitive — never committed)

| Variable                       | Description                          |
|--------------------------------|--------------------------------------|
| `vyos_user_encrypted_password` | SHA-512 crypt hash for the vyos user |
| `vyos_wg0_private_key`         | WireGuard private key for wg0        |
| `vyos_wg0_peer_public_key`     | VPS WireGuard public key             |

---

## Interfaces

| Interface    | Address           | Description                                    |
|--------------|-------------------|------------------------------------------------|
| `eth0`       | DHCP              | WAN uplink (Home ISP)                          |
| `eth1`       | —                 | 802.1Q trunk to Proxmox host                   |
| `eth1.100`   | `10.10.100.1/29`  | VLAN 100 — DMZ                                 |
| `eth1.200`   | `10.10.200.1/24`  | VLAN 200 — SERVERS                             |
| `wg0`        | `10.255.255.2/29` | WireGuard tunnel to OVH VPS                    |
| `lo`         | —                 | Loopback (reserved for router services)        |

All physical interfaces have GRO, GSO, SG, and TSO offload enabled.

---

## WireGuard (wg0)

| Parameter          | Value                  |
|--------------------|------------------------|
| Local address      | `10.255.255.2/29`      |
| Listen port        | `51820`                |
| Peer endpoint      | `51.79.161.145:51830`  |
| Peer allowed-ips   | `0.0.0.0/0`            |
| Persistent keepalive | `25s`               |

Private key and peer public key are stored in `vyos-vault.yml`.

---

## DHCP

Single shared network `SERVERS` on `10.10.200.0/24`.

| Parameter      | Value                                    |
|----------------|------------------------------------------|
| Subnet         | `10.10.200.0/24`                         |
| Range          | `10.10.200.101` – `10.10.200.200`        |
| Default router | `10.10.200.1`                            |
| Domain name    | `magbanua.xyz`                           |
| DNS servers    | `1.1.1.1`, `8.8.8.8`, `9.9.9.9`         |
| Lease time     | `86400s` (24 h)                          |

---

## NTP

Servers: `time1.vyos.net`, `time2.vyos.net`, `time3.vyos.net`

Clients allowed: `127.0.0.0/8`, `169.254.0.0/16`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, and IPv6 loopback/link-local/ULA ranges.

---

## SSH

| Listen address   | Reachable from                    |
|------------------|-----------------------------------|
| `10.10.200.1`    | VLAN 200 (SERVERS)                |
| `172.16.1.182`   | Proxmox host LAN                  |
| `192.168.8.182`  | Home LAN                          |

---

## Static Routes

| Prefix | Next Hop | Description |
|---|---|---|
| `10.200.10.0/24` | `10.10.200.15` | wp-host1 container subnet |
| `172.16.100.0/24` | `10.255.255.2` | Remote subnet behind OVH VPS (via wg0) |
| `172.16.196.0/24` | `10.255.255.2` | Remote OVH OPNsense network (via wg0) |

To add a new route, append to `vyos_static_routes` in `vars/main.yml`:

```yaml
vyos_static_routes:
  - prefix: 10.99.0.0/24
    next_hop: 10.10.200.1
    description: "New remote network"
```

Then apply with `--tags routes`.

---

## Notes

- `hw-id` (MAC address) fields are intentionally absent — VyOS sets these from hardware and rejects `set` commands that attempt to change them.
- `save: true` on each task commits and persists the config to startup. Re-running the playbook when config is already in sync results in no changes.
- The WireGuard private key is written to the router in plaintext (as required by VyOS). Ensure the Ansible control host is trusted and `/opt/infra/secrets/` has appropriate file permissions.
- `commit-revisions 100` keeps the last 100 config snapshots on the router — roll back with `rollback <n>` from the VyOS CLI.
