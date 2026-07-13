# odroid-blue

## Overview

Purpose-built Linux server running on the former Home Assistant Blue (Odroid N2+).

Current role: Docker host, Linux learning environment, development server, and homelab utility machine.

---

## Hardware

| Item      | Value                            |
| --------- | -------------------------------- |
| Hostname  | odroid-blue                      |
| Platform  | Odroid N2+ (Home Assistant Blue) |
| OS        | DietPi                           |
| Storage   | eMMC                             |
| Network   | DHCP                             |
| Tailscale | Enabled                          |

---

## Network Information

### Local Access

| Service         | URL                      |
| --------------- | ------------------------ |
| Cockpit         | https://192.168.0.9:9090 |
| Portainer       | https://192.168.0.9:9443 |
| Nginx Test Site | http://192.168.0.9:8080  |
| phpMyAdmin      | http://192.168.0.9:8081  |
| Samba Share     | \\192.168.0.9\mark       |

### Tailscale

| Item         | Value                         |
| ------------ | ----------------------------- |
| Hostname     | odroid-blue                   |
| Tailnet Name | odroid-blue.taila257ea.ts.net |
| Tailscale IP | 100.103.76.93                 |

---

## Installed Services

### Cockpit

Linux administration web interface.

URL:

https://192.168.0.9:9090

Capabilities:

* System monitoring
* Terminal access
* Service management
* Log viewing
* Software updates

---

### Portainer

Docker management interface.

URL:

https://192.168.0.9:9443

Capabilities:

* Container management
* Image management
* Compose stacks
* Docker logs
* Volumes and networks

---

### Samba

Provides Windows file access.

Share:

\\192.168.0.9\mark

Primary use:

* Edit Docker projects from Windows
* Open projects directly in VS Code

---

### Tailscale

Secure remote networking.

Capabilities:

* Remote access
* MagicDNS
* Secure SSH
* Access to services without port forwarding

---

## Docker Projects

Root directory:

/home/mark/docker

Current projects:

### nginx-mariadb

Location:

/home/mark/docker/nginx-mariadb

Components:

* Nginx
* PHP-FPM
* MariaDB
* phpMyAdmin

Ports:

| Service    | Port |
| ---------- | ---- |
| Nginx      | 8080 |
| phpMyAdmin | 8081 |
| MariaDB    | 3306 |

Directory structure:

/home/mark/docker/nginx-mariadb
├── docker-compose.yml
├── Dockerfile.php
├── html
├── html-local
├── mariadb_data
└── nginx

#### Remote vs. Local project URLs

Two nginx server blocks in `nginx/conf.d/` route to the same host, split by domain:

| | **Remote** (`default.conf`) | **Local** (`local.conf`) |
| --- | --- | --- |
| Domain | `*.windomlane.org` | `*.192-168-0-9.sslip.io` |
| Reachable via | Cloudflare Tunnel wildcard rule → `localhost:8080` | direct LAN connection, no tunnel |
| Host port → container port | `8080:80` | `80:8082` |
| Docroot | `/var/www/html/$project` (`./html`) | `/var/www/html-local/$project` (`./html-local`) |
| PHP support | Yes (fastcgi → `php:9000`) | No, static files only |

Both use an nginx `map` block that captures the subdomain via regex and injects it into `root` as `$project`, so a new folder dropped into `./html` or `./html-local` instantly gets its own subdomain with no nginx reload needed.

**sslip.io** is a public wildcard DNS service: any hostname shaped like `<anything>.<IP-with-dashes>.sslip.io` resolves to that embedded IP. `foo.192-168-0-9.sslip.io` always resolves to `192.168.0.9`, so it only actually works from the LAN (the IP is private) — no local DNS server or `/etc/hosts` edits required to get automatic per-project local subdomains.

* `foo.windomlane.org` → public internet, PHP-capable, served from `./html/foo`
* `foo.192-168-0-9.sslip.io` → LAN-only, static-only, served from `./html-local/foo`

---

## Home Assistant Environment

Primary Home Assistant server:

Pulcro N150 Mini PC

Remote access:

Cloudflare Tunnel

URL:

https://ha.windomlane.org

Notes:

* Nginx Proxy Manager no longer required
* Legacy port forwarding on 80/443 removed
* Cloudflare Tunnel handles external access

---

## Future Projects

Potential additions:

* Gitea
* Wiki.js
* Uptime Kuma
* Grafana
* InfluxDB
* MQTT services
* Development environments

---

## Troubleshooting Notes

### Cockpit Limited Access Mode

Symptoms:

* Warning banner in Cockpit
* Reduced functionality

Resolution:

* D-Bus was not running
* Installed/configured polkit
* Restarted Cockpit

Status:

Resolved

---

### PHP Database Driver

Symptoms:

Connection failed: could not find driver

Cause:

pdo_mysql extension not installed in PHP container.

Status:

Pending
