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

Two plays: `k8s_common` applied to all nodes, then `k8s_control` applied to the control plane only.

```bash
ansible-playbook playbooks/k8s_lab/setup-k8s-lab.yml -i inventory/lab/hosts_k8s_lab
```

---

## Roles

### `k8s_common`

Applied to all k8s lab nodes (control plane and workers):

- Adds all three node hostnames to `/etc/hosts` on each node
- Disables swap (`swapoff -a`), comments out swap entries in `/etc/fstab`, and masks `swap.target`
- Writes `/etc/modules-load.d/k8s.conf` with `overlay` and `br_netfilter`
- Loads `overlay` and `br_netfilter` kernel modules immediately
- Writes `/etc/sysctl.d/k8s.conf` and applies sysctl parameters for k8s networking (`bridge-nf-call-iptables`, `bridge-nf-call-ip6tables`, `ip_forward`)
- Installs containerd (version controlled by `containerd_version` default, currently `2.1.4`): downloads and extracts binaries to `/usr/local`, installs the upstream systemd unit, enables and starts the service
- Installs `runc` via apt
- Adds the Kubernetes apt repository (`pkgs.k8s.io`, stable v1.34)
- Installs and holds `kubelet`, `kubeadm`, and `kubectl`
- Writes `/etc/sysctl.d/99-nodeport.conf` and sets `net.ipv4.ip_local_port_range = 49152 65535` (frees ports below 49152 for k8s NodePorts)
- Creates `/etc/kubernetes/patches/` and writes `kubeletconfiguration0+strategic.yaml` — a kubeadm patch that sets shutdown grace periods, `kubeReserved`/`systemReserved` CPU+memory+ephemeral-storage, and hard eviction thresholds

Defaults: `containerd_version: "2.1.4"`, `containerd_arch: "amd64"`.

### `k8s_control`

Applied to `k8s_control_plane` group only. Control-plane-specific tasks (kubeadm init and beyond) go here.
