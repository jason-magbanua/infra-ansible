# k8s Lab

External k8s lab cluster (3 nodes, public IPs). Managed separately from the Proxmox homelab.

---

## Inventory

`inventory/lab/hosts_k8s_lab` — static, not Terraform-generated.

| Host                  | IP              | Role          |
|-----------------------|-----------------|---------------|
| jm-k8s-control        | 109.61.95.136   | Control plane |
| jm-k8s-lin-worker1    | 109.61.95.137   | Worker        |
| jm-k8s-lin-worker2    | 109.61.95.138   | Worker        |

SSH key: `~/.ssh/id_rsa_ansible`, user: `root`.

---

## Playbooks

### `k8s_lab/setup-k8s-lab.yml`

Bootstraps all k8s lab nodes via the `k8s_common` role.

```bash
ansible-playbook playbooks/k8s_lab/setup-k8s-lab.yml -i inventory/lab/hosts_k8s_lab
```

---

## Roles

### `k8s_common`

Baseline applied to all k8s lab nodes:

- Adds all three node hostnames to `/etc/hosts` on each node
