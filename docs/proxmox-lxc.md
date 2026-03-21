# LXD Container Workloads

Playbooks for managing LXD containers on VM hosts. All playbooks run on `localhost` and issue `lxc` CLI commands against a named LXD remote — no SSH into the containers directly.

---

## Playbooks

| Playbook                 | Purpose                                                    | Run order |
|--------------------------|------------------------------------------------------------|-----------|
| `setup-lemp.yml`         | Create a container and install Nginx + MariaDB + PHP       | 1st       |
| `install-wordpress.yml`  | Deploy WordPress on an existing LEMP container             | 2nd       |
| `performance-tuning.yml` | Apply adaptive Nginx + PHP-FPM tuning (OPcache, FastCGI)  | 3rd       |

---

## 1. LEMP Setup — `setup-lemp.yml`

Provisions a new LXD container on the `wp-host1` remote and installs a full LEMP stack (Nginx, MariaDB, PHP 8.3). Idempotent — skips container creation if it already exists.

### Required variables

| Variable         | Description                                 |
|------------------|---------------------------------------------|
| `container_name` | Name of the container to create or target   |

### Optional variables

| Variable           | Default      | Description                          |
|--------------------|--------------|--------------------------------------|
| `lxd_remote`       | `wp-host1`   | LXD remote name                      |
| `container_image`  | `ubuntu-24.04` | Image alias (pre-loaded by setup-lxc.yml) |
| `container_cpu`    | `1`          | vCPU limit                           |
| `container_memory` | `512MB`      | Memory limit                         |
| `container_disk`   | `5GB`        | Root disk size                       |
| `php_version`      | `8.3`        | PHP version to install               |

### What it does

1. Creates and starts the container if it does not exist
2. Applies CPU, memory, and disk limits
3. Waits for cloud-init to complete
4. Installs: `nginx`, `mariadb-server`, `php8.3`, `php8.3-fpm`, `php8.3-mysql`, `php8.3-curl`, `php8.3-gd`, `php8.3-mbstring`, `php8.3-xml`, `php8.3-zip`

### Usage

```bash
ansible-playbook playbooks/LXC/setup-lemp.yml \
  -e container_name=mysite

# Override resources
ansible-playbook playbooks/LXC/setup-lemp.yml \
  -e container_name=mysite \
  -e container_cpu=2 \
  -e container_memory=1GB \
  -e container_disk=10GB
```

---

## 2. WordPress — `install-wordpress.yml`

Deploys WordPress into an existing LEMP container. Idempotent — detects an existing `wp-config.php` and skips download/install steps on re-runs, only updating config files.

### Required variables

| Variable         | Description                                          |
|------------------|------------------------------------------------------|
| `container_name` | Target container name                                |
| `domain`         | Domain name (used for nginx vhost and wp-config.php) |

### Optional variables

| Variable       | Default        | Description                  |
|----------------|----------------|------------------------------|
| `lxd_remote`   | `wp-host1`     | LXD remote name              |
| `php_version`  | `8.3`          | PHP version                  |
| `wp_root`      | `/var/www/html`| WordPress document root      |

### What it does

**Fresh install:**
1. Downloads and extracts WordPress to `wp_root`
2. Generates random DB name, user, and password
3. Saves credentials to `/opt/infra/secrets/<container_name>-db.yml` (mode `0600`)
4. Creates the MariaDB database and user
5. Renders and pushes `wp-config.php` from template

**Every run:**
- Re-renders and pushes `wp-config.php` and the Nginx vhost config
- Tests and reloads Nginx
- Prints a deployment summary

### DB credential management

Credentials are auto-generated on first install and persisted at:

```
/opt/infra/secrets/<container_name>-db.yml
```

On re-runs, this file is loaded and used to re-render `wp-config.php`. Do not delete this file unless you are also dropping and recreating the database.

### HTTPS behind a reverse proxy

`wp-config.php` sets `WP_HOME` / `WP_SITEURL` to `https://{{ domain }}` and trusts the `X-Forwarded-Proto` header from the upstream proxy (e.g. Nginx Proxy Manager). This prevents mixed-content errors when SSL is terminated upstream.

### Usage

```bash
ansible-playbook playbooks/LXC/install-wordpress.yml \
  -e container_name=mysite \
  -e domain=mysite.example.com
```

---

## 3. Performance Tuning — `performance-tuning.yml`

Applies adaptive Nginx and PHP-FPM tuning based on the container's actual vCPU count and RAM. Works for any LEMP container, not just WordPress.

### Required variables

| Variable         | Description            |
|------------------|------------------------|
| `container_name` | Target container name  |
| `domain`         | Domain name            |

### Optional variables

| Variable       | Default      | Description          |
|----------------|--------------|----------------------|
| `lxd_remote`   | `wp-host1`   | LXD remote name      |
| `php_version`  | `8.3`        | PHP version          |
| `wp_root`      | `/var/www/html` | WordPress root    |

### Tuning matrix

`php_pm_max_children` is selected based on the live vCPU + RAM values read from the container:

| vCPUs | RAM       | `pm.max_children` | `nginx worker_processes` |
|-------|-----------|-------------------|--------------------------|
| 1     | ≤ 600 MB  | 5                 | 1                        |
| 1     | ≤ 1200 MB | 8                 | 1                        |
| 2     | ≤ 2500 MB | 15                | auto                     |
| 2     | > 2500 MB | 25                | auto                     |
| other | any       | 6                 | 1                        |

### What it does

1. Reads live vCPU count (`nproc`) and RAM (`/proc/meminfo`) from the container
2. Installs `php8.3-opcache` and pushes `files/opcache.ini`
3. Creates `/var/cache/nginx` for FastCGI page caching
4. Renders and pushes the cached WordPress Nginx vhost (`nginx-vhost-cached.conf.j2`)
5. Replaces the default PHP-FPM pool with a tuned `www.conf` (`php-fpm-www.conf.j2`)
6. Renders and pushes a tuned global `nginx.conf` (`nginx-adaptive.conf.j2`)
7. Restarts PHP-FPM and reloads Nginx

### Usage

```bash
ansible-playbook playbooks/LXC/performance-tuning.yml \
  -e container_name=mysite \
  -e domain=mysite.example.com
```

---

## Template Reference

| Template                      | Used by               | Purpose                                  |
|-------------------------------|-----------------------|------------------------------------------|
| `nginx-vhost-basic.conf.j2`   | `install-wordpress`   | Basic Nginx vhost (no caching)           |
| `nginx-vhost-cached.conf.j2`  | `performance-tuning`  | Nginx vhost with FastCGI page cache      |
| `nginx-adaptive.conf.j2`      | `performance-tuning`  | Global nginx.conf scaled to worker count |
| `php-fpm-www.conf.j2`         | `performance-tuning`  | PHP-FPM pool tuned to `pm.max_children`  |
| `wp-config.php.j2`            | `install-wordpress`   | WordPress config with DB creds + proxy   |
