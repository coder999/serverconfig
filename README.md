# serverconfig

Documentation and config notes for Mark's home lab and household infrastructure.

## Network Overview

| Host | LAN IP | Tailscale IP | Role |
| ---- | ------ | ------------- | ---- |
| odroid-blue | 192.168.0.9 | 100.103.76.93 | DietPi homeserver — Cloudflare tunnel ingress, Docker stack, Cockpit |
| Home Assistant host | 192.168.0.100 | — | HA Core/Supervisor, Scrypted (camera pipeline), MQTT broker, own Cloudflare tunnel |
| raspberrypi | DHCP (see [pi.md](pi.md)) | — | SSH admin, Samba, Docker, Portainer, Cockpit |
| CenturyLink/Lumen C4000BG | gateway | — | ISP router/gateway (OpenWrt-based) |

odroid-blue and the Home Assistant host are **separate machines**, each running its own independent Cloudflare tunnel — they share nothing but the LAN, so an incident on one should never be assumed to explain an incident on the other without checking logs from both sides.

## Files

- **[odroid.md](odroid.md)** — odroid-blue setup: Cloudflare tunnel config (`*.windomlane.org` routing), Docker stack (nginx/mariadb/phpmyadmin/portainer), Cockpit proxy, known issues and fixes.
- **[pi.md](pi.md)** — raspberrypi host details: SSH access, Samba, Docker workloads, Portainer, Cockpit.
- **[century_link_router.md](century_link_router.md)** — CenturyLink/Lumen C4000BG gateway: hardware, OpenWrt build, network/WAN config, SSH access notes.
- **[appliance_database.md](appliance_database.md)** — household appliance inventory (make/model/serial) for the property, unrelated to server infra but kept here for reference.

## Key Points to Know

- **Cloudflare tunnel (odroid-blue)**: config at `/etc/cloudflared/config.yml`, tunnel ID `28d72c9e-2591-4bcc-8848-e08616ce6f71`. Routes `nvr`, `cameras`, `sandbox`, `odroid` subdomains plus a wildcard fallback to the local nginx container. **Not** how Home Assistant is exposed — `ha.windomlane.org` is served by the HA host's own independent `cloudflared` add-on.
- **Camera pipeline**: cameras route through **Scrypted** on the Home Assistant host (192.168.0.100). odroid-blue also runs `go2rtc`, but it is *not* part of the active camera pipeline — don't assume it's relevant when debugging camera issues.
- **odroid-blue resource constraint**: only 3.7GB RAM + 1.8GB swap, and it's also where Claude Code background sessions run. Concurrent sessions have previously triggered OOM kills that disrupted the Cloudflare tunnel — check `free -h` / `ps -eo rss,comm --sort=-rss` if cloudflared or Docker misbehave.
- **Cockpit auth on odroid.windomlane.org**: if login fails, check `systemctl is-active cockpit-session-socket-guard.timer` first — this guard (added 2026-07-01) keeps `cockpit-session.socket` alive, working around `cockpit-tls` tearing it down after idle.
- **Sudo**: `mark` has full passwordless sudo on odroid-blue (`/etc/sudoers.d/mark-nopasswd`, includes `Defaults:mark !requiretty` so it also works from non-TTY/background contexts).

## Conventions

- `.claude/` is excluded from this repo via `.gitignore`.
