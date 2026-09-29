# OpenWrt monitoring

`roles/openwrt_node_exporter` installs `prometheus-node-exporter-lua` on the OpenWrt home router (openwrt1, 172.16.1.1) and 2nd-floor AP (openwrt2, 172.16.1.2) so Prometheus can scrape them. They are scraped as `job="node_exporter"` with `instance` set to the host name, so they appear in the existing Grafana dashboards.

## Playbook

```bash
cd /opt/infra/ansible
/opt/automation/ansible-venv/bin/ansible-playbook playbooks/openwrt/setup-monitoring.yml [--limit openwrt2]
```

Re-run it after a sysupgrade (packages are not kept).

## Inventory

`[openwrt]` group in `inventory/lab/hosts`, connecting as `root` with `~/.ssh/root-sshkey.rsa`. `site.yml` excludes the group (`all:!vyos:!openwrt`): OpenWrt has no apt and no Python, so the `common` and `node_exporter` roles don't apply.

## Role

OpenWrt has no Python, so every task uses `ansible.builtin.raw`, guarded to stay idempotent:

| Task | Guard |
|---|---|
| `opkg install prometheus-node-exporter-lua` | skipped when `opkg list-installed` shows it |
| `uci set` of `listen_interface` / `listen_port`, then restart | skipped when the UCI values already match |
| enable at boot, start | only acts when not enabled / not listening |

| Variable | Default | Description |
|---|---|---|
| `openwrt_node_exporter_package` | `prometheus-node-exporter-lua` | opkg package |
| `openwrt_node_exporter_port` | `9100` | Listen port |
| `openwrt_node_exporter_interface` | `lan` | UCI network the exporter binds to |

## Scrape targets

`openwrt1` and `openwrt2` are in `node_exporter_targets` (`roles/prometheus/defaults/main.yml`). Re-run `playbooks/LXC/setup-prometheus.yml` after changing it.

## Limits

The Lua exporter has no filesystem, systemd or pressure collectors: CPU, memory, load, network, conntrack, uptime and `node_uname_info` work; disk panels stay empty.
