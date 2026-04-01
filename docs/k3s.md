# k3s Cluster

Lightweight Kubernetes (k3s) on three VMs provisioned by Terraform.

---

## Hosts

| Host          | IP              | Role           |
|---------------|-----------------|----------------|
| `k3s-control` | 10.10.200.60    | Server (control plane) |
| `k3s-worker1` | 10.10.200.61    | Agent          |
| `k3s-worker2` | 10.10.200.62    | Agent          |

All hosts are in the `proxmox_vms` inventory group (user: `infra`).

---

## NFS Data Mount

All nodes mount the Proxmox NFS export for shared persistent storage:

| Item       | Value                              |
|------------|------------------------------------|
| Source     | `172.16.1.8:/mnt/ssd_1T/k3s-data` |
| Mount      | `/mnt/k3s-data`                    |
| Options    | `defaults,_netdev,nofail`          |

The `_netdev` option defers mount until networking is up. `nofail` prevents boot failures if the NFS server is unreachable.

---

## Role: `k3s`

| Variable         | Default              | Description                  |
|------------------|----------------------|------------------------------|
| `k3s_version`    | `v1.32.3+k3s1`       | k3s release to install       |
| `k3s_role`       | `agent`              | `server` or `agent`          |
| `k3s_server_ip`  | `10.10.200.60`       | Control plane IP             |
| `k3s_server_port`| `6443`               | API server port              |
| `k3s_nfs_server` | `172.16.1.8`         | NFS server IP                |
| `k3s_nfs_export` | `/mnt/ssd_1T/k3s-data` | NFS export path            |
| `k3s_nfs_mount`  | `/mnt/k3s-data`      | Local mount point            |

Task files:
- `tasks/main.yml` — nfs-common install, NFS mount, delegates to server/agent
- `tasks/server.yml` — k3s server install via install script, waits for node token
- `tasks/agent.yml` — fetches token from control plane, installs k3s agent

---

## Playbook

```bash
# Full cluster install (control first, then workers)
ansible-playbook playbooks/VMs/setup-k3s.yml

# Control plane only (workers need the server token first)
ansible-playbook playbooks/VMs/setup-k3s.yml --limit k3s-control

# Workers only (after control plane is up)
ansible-playbook playbooks/VMs/setup-k3s.yml --limit 'k3s-worker*'
```

---

## Post-install

Check cluster from the control node:

```bash
sudo kubectl get nodes
```

The kubeconfig is at `/etc/rancher/k3s/k3s.yaml` on `k3s-control`. Copy to `~/.kube/config` and update the server address to use remotely.

---

## Pitfalls

- Workers fetch the node token via `delegate_to: k3s-control` — the control plane must be up and the token file present before running agents.
- k3s install script uses `creates: /usr/local/bin/k3s` for idempotency — re-running won't reinstall. To upgrade, remove the binary first or set a new `k3s_version` and uninstall with `/usr/local/bin/k3s-uninstall.sh`.
- NFS mount requires `nfs-common` — the role installs it before mounting.
