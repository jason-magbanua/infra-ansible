# VyOS Router Configuration

Automates the full VyOS router configuration via the `vyos.vyos` Ansible collection. The playbook is idempotent — re-running it applies only what has drifted from the declared state.

---

## Network Overview

```
Internet
   │
  eth0 (DHCP) ──── transit0, Home ISP uplink
   │
 VyOS (172.16.1.182 / 192.168.8.182)
   │
  wg0 ──────────── 10.255.255.2/29 — WireGuard overlay to VPS (primary egress)
   │
  eth1 ─────────── 802.1Q trunk to Proxmox host
    ├── vlan100 ── 10.10.100.1/29 — DMZ (public services)
    └── vlan200 ── 10.10.200.1/27 — SERVERS (internal)
```

Policy-based routing (PBR) sends all egress from VLAN100/VLAN200 through `wg0`. The `prerouting raw` firewall rules exempt local inter-VLAN traffic and known private subnets from PBR.

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
ansible-playbook playbooks/vyos/vyos.yml --tags firewall
ansible-playbook playbooks/vyos/vyos.yml --tags interfaces,routes
```

---

## Tags Reference

| Tag          | What it configures                                             |
|--------------|----------------------------------------------------------------|
| `system`     | Hostname, timezone, commit revisions, console, syslog         |
| `auth`       | Login user, encrypted password, SSH public key                |
| `interfaces` | eth0 (WAN), eth1 (trunk), VLAN 100/200 subinterfaces, wg0    |
| `firewall`   | IPv4 forward filter rules + prerouting raw rules (PBR)        |
| `dhcp`       | DHCP server for the SERVERS subnet (10.10.200.0/27)           |
| `ntp`        | NTP servers and allowed client addresses                      |
| `ssh`        | SSH listen addresses                                          |
| `routes`     | Static routes                                                 |

---

## Variables

### `vars/main.yml` (non-sensitive)

Key variables — see the file for the full list.

| Variable                    | Description                                  |
|-----------------------------|----------------------------------------------|
| `vyos_hostname`             | Router hostname (`vyos`)                     |
| `vyos_timezone`             | System timezone (`Asia/Manila`)              |
| `vyos_vlan100_address`      | DMZ gateway IP (`10.10.100.1/29`)            |
| `vyos_vlan200_address`      | SERVERS gateway IP (`10.10.200.1/27`)        |
| `vyos_wg0_address`          | WireGuard tunnel address (`10.255.255.2/29`) |
| `vyos_dhcp_network_name`    | DHCP shared network name (`SERVERS`)         |
| `vyos_dhcp_subnet`          | DHCP subnet (`10.10.200.0/27`)               |
| `vyos_dhcp_range_start/stop`| DHCP range (`.11` – `.30`)                  |
| `vyos_ssh_listen_addresses` | SSH bind addresses                           |
| `vyos_static_routes`        | List of `{prefix, next_hop, description}`    |

### `/opt/infra/secrets/vyos-vault.yml` (sensitive — never committed)

| Variable                       | Description                          |
|--------------------------------|--------------------------------------|
| `vyos_user_encrypted_password` | SHA-512 crypt hash for the vyos user |
| `vyos_wg0_private_key`         | WireGuard private key for wg0        |
| `vyos_wg0_peer_address`        | VPS public IP (endpoint)             |
| `vyos_wg0_peer_public_key`     | VPS WireGuard public key             |

---

## Firewall Logic

### IPv4 Forward Filter

Controls traffic routed between interfaces.

| Rule | Action | Description                                |
|------|--------|--------------------------------------------|
| 1    | accept | Established/related connections            |
| 10   | accept | Home LAN (172.16.0.0/12) SSH → DMZ port 22|
| 11   | accept | Home LAN ping → DMZ                       |
| 20   | drop   | DMZ SSH → SERVERS blocked                 |
| 28   | accept | NPM (10.10.100.2) → pve1 (10.10.200.8)    |
| 29   | accept | NPM (10.10.100.2) → streaming-aio          |
| 30   | drop   | DMZ → Home LAN blocked                    |
| 40   | drop   | DMZ → 192.168.0.0/16 blocked              |

### IPv4 Prerouting Raw (PBR exclusions)

Packets that hit `accept` in this chain bypass policy-based routing and use the normal routing table. This keeps inter-VLAN and local traffic off the WireGuard tunnel.

| Rule | Exempt traffic                            |
|------|-------------------------------------------|
| 1–2  | DMZ ↔ SERVERS inter-VLAN                |
| 3–4  | Same-subnet traffic within each VLAN      |
| 5–6  | SERVERS/DMZ → 172.16.0.0/12 (home LAN)   |
| 7–8  | SERVERS/DMZ → 192.168.0.0/16             |

---

## Adding Firewall Rules

Edit the `lines` list in the relevant task in `vyos.yml`:

```yaml
- "set firewall ipv4 forward filter rule 50 action accept"
- "set firewall ipv4 forward filter rule 50 description 'My new rule'"
- "set firewall ipv4 forward filter rule 50 destination address 10.10.200.5/32"
- "set firewall ipv4 forward filter rule 50 source address 10.10.100.0/29"
```

Then apply with `--tags firewall`.

---

## Adding Static Routes

Add an entry to `vyos_static_routes` in `vars/main.yml`:

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
