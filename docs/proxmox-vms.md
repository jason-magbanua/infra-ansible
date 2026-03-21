# Proxmox VM Host Setup

Playbooks for bootstrapping VM hosts provisioned by Terraform. Run these once after a new VM is online. Both playbooks target the `proxmox_vms` inventory group.

---

## Playbooks

| Playbook         | Purpose                                              |
|------------------|------------------------------------------------------|
| `setup-zfs.yml`  | Create a ZFS mirror pool on two attached data disks  |
| `setup-lxc.yml`  | Install and configure LXD on top of the ZFS pool     |

---

## ZFS Setup — `setup-zfs.yml`

Creates a mirrored ZFS pool named `user-data` across two attached data disks and applies tuning appropriate for a low-memory homelab VM.

### Variables

| Variable            | Default        | Description                                   |
|---------------------|----------------|-----------------------------------------------|
| `zfs_pool_name`     | `user-data`    | Name of the ZFS pool                          |
| `zfs_mountpoint`    | `/user-data`   | Mount path for the pool                       |
| `zfs_arc_max_bytes` | `1073741824`   | ZFS ARC size cap in bytes (default: 1 GB)     |
| `zfs_disks`         | `[/dev/sdb, /dev/sdc]` | Block devices for the mirror          |

### What it does

1. Installs `zfsutils-linux`
2. Caps the ARC cache via `/etc/modprobe.d/zfs.conf` and triggers `update-initramfs`
3. Creates the pool if it does not exist:
   - `ashift=12` — 4K sector alignment
   - `compression=lz4` — transparent compression
   - `atime=off` — no access time writes
   - `xattr=sa`, `acltype=posixacl` — POSIX ACL support for Linux containers

### Usage

```bash
ansible-playbook playbooks/VMs/setup-zfs.yml
```

Override disk devices or ARC limit for a specific host:

```bash
ansible-playbook playbooks/VMs/setup-zfs.yml \
  -e zfs_arc_max_bytes=2147483648 \
  -e '{"zfs_disks": ["/dev/vdb", "/dev/vdc"]}'
```

---

## LXD Setup — `setup-lxc.yml`

Installs LXD (via snap), creates a ZFS-backed storage pool, configures a bridge network, and pre-loads the Ubuntu 24.04 container image.

### Variables

| Variable              | Default          | Description                              |
|-----------------------|------------------|------------------------------------------|
| `lxd_storage_pool`    | `lxdpool`        | Name of the LXD storage pool            |
| `zfs_dataset`         | `user-data`      | ZFS pool to back the storage pool        |
| `ubuntu_image_alias`  | `ubuntu-24.04`   | Local alias for the cached image         |
| `lxd_network`         | `lxdbr0`         | LXD bridge network name                  |
| `lxd_network_ipv4`    | `10.200.10.1/24` | Bridge IP and subnet for containers      |

### What it does

1. Installs `snapd` and `lxd` snap, then waits for the daemon to be ready
2. Creates the `lxdpool` ZFS storage pool (backed by the `user-data` dataset) if absent
3. Attaches the storage pool as the root disk in the default LXD profile
4. Creates the `lxdbr0` bridge network with NAT and IPv6 disabled
5. Attaches the bridge to the default profile's `eth0`
6. Downloads the Ubuntu 24.04 image from the public LXD image server

Pass `--extra-vars debug_output=true` to print storage pool, network, and image state at the end.

### Usage

```bash
ansible-playbook playbooks/VMs/setup-lxc.yml
ansible-playbook playbooks/VMs/setup-lxc.yml -e debug_output=true
```

---

## Connecting the Infra Server to an LXD Remote

Before Ansible can manage containers on a VM host, the infra server's `lxc` client must be trusted by that host's LXD daemon. LXD does not listen on the network by default.

**1. Enable remote API on the VM host:**

```bash
lxc config set core.https_address :8443
```

**2. Generate a one-time trust token on the VM host:**

```bash
lxc config trust add --name infra-server
```

**3. Add the remote on the infra server using the printed token:**

```bash
lxc remote add wp-host1 https://<vm-host-ip>:8443 --token <token>
```

**4. Verify:**

```bash
lxc remote list
lxc ls wp-host1:
```

The token is single-use and short-lived — run all four steps in the same session.
