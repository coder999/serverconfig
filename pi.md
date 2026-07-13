# raspberrypi

## Overview

Primary Raspberry Pi host used for SSH administration, Samba file sharing, Docker workloads, Portainer, and Cockpit.

## Host Details

| Item | Value |
| ---- | ----- |
| Hostname | raspberrypi |
| Primary user | mark |
| SSH key file (Windows client) | ~/.ssh/id_ed25519_rpi19 |
| SSH alias (Windows client) | rpi-19 |
| Network | DHCP |

## Network Information

Current LAN IP:

- 192.168.0.27

Notes:

- This host was previously reachable at 192.168.0.19 and later moved to 192.168.0.27.
- SSH alias rpi-19 on Windows was updated to point to 192.168.0.27.

## Access URLs

| Service | URL |
| ------- | --- |
| Portainer | https://192.168.0.27:9443 |
| Cockpit | https://192.168.0.27:9090 |
| Samba share | \\192.168.0.27\rootfs |

## Installed Services

### Docker

- Package: docker.io
- Verified active via systemd
- Verified running containers with docker ps

### Portainer

- Container name: portainer
- Image: portainer/portainer-ce:lts
- Restart policy: always
- Ports: 8000/tcp and 9443/tcp

### Cockpit

- Packages: cockpit, cockpit-bridge, cockpit-ws, cockpit-system, cockpit-packagekit, cockpit-networkmanager, cockpit-storaged
- Service socket: cockpit.socket (active)
- Web UI: https://192.168.0.27:9090

### Samba

- Package: samba
- Services: smbd and nmbd (enabled, active)
- Share name: rootfs
- Path: /
- Access: read-only
- Valid users: mark

### RTL-AMR

- Core services: rtl-tcp, rtlamr-mqtt, meter-health-mqtt
- rtl-tcp endpoint: 127.0.0.1:1234 (local only)
- MQTT base topic: home/meters
- Home Assistant discovery prefix: homeassistant

Web interface options:

- No built-in native web UI in rtlamr itself.
- Home Assistant MQTT entities are the primary dashboard view.
- MQTT Explorer can be used from Windows to inspect raw topics.
- Cockpit can be used for service status and logs.

Current status snapshot:

- rtl-tcp.service: active
- meter-health-mqtt.service: active
- rtlamr-mqtt.service: active

Troubleshooting note (2026-06-13):

- Symptom: rtlamr-mqtt was in a restart loop with `read tcp 127.0.0.1:*->127.0.0.1:1234: i/o timeout`.
- Root cause: rtl-tcp/USB SDR stream instability during service restarts.
- Fix applied: restarted `rtl-tcp.service` and `rtlamr-mqtt.service`, verified `usbcore.usbfs_memory_mb=0` at runtime and in `/boot/cmdline.txt`.
- Hardening added: systemd drop-in `/etc/systemd/system/rtlamr-mqtt.service.d/wait-for-rtl-tcp.conf` with `ExecStartPre` wait loop for `127.0.0.1:1234`.
- Validation: rtlamr-mqtt stayed running and published meter data again to MQTT topics.

Candidate meter review (target read 2857500):

- 14 unique meter IDs discovered.
- No currently discovered ID is close to 2857500.
- Closest active IDs at this time were:
	- 53356641 (about 857164)
	- 16367702 (about 331977)
- This strongly suggests the desired meter ID has not been observed yet, or the expected reading scale does not match RTL-AMR `Consumption` units for the target meter type.

Useful checks:

- sudo systemctl status rtl-tcp.service rtlamr-mqtt.service meter-health-mqtt.service --no-pager
- sudo journalctl -u rtlamr-mqtt.service -n 100 --no-pager
- sudo journalctl -u rtl-tcp.service -n 100 --no-pager

## Operational Notes

- For richer RTL-AMR visualization, add a metrics stack (for example, MQTT -> Telegraf/Node-RED -> InfluxDB -> Grafana).

## Next Session Checklist

1. Verify service stability first:
	- `ssh rpi-19 "systemctl show rtlamr-mqtt.service -p ActiveState -p SubState -p NRestarts -p ExecMainStatus; echo ---; systemctl show rtl-tcp.service -p ActiveState -p SubState -p ExecMainStatus"`
2. Confirm last-24h capture volume:
	- Run `/tmp/amr_24h_stats.py` style check and record entry count + unique IDs.
3. Rank current candidates against target read 2857500:
	- Run `/tmp/amr_24h_rank.py` style check and keep top 10 by smallest diff.
4. Success criteria:
	- Services active/running, restarts stable, healthy 24h message count, and at least one plausible candidate with repeated samples and positive span.
5. If unsuccessful after stable 24h:
	- Run gain sweep (auto, 14, 28, 42), frequency sweep near 912.6 MHz, then compare capture volume + closest-candidate ranking.

## Sweep Progress Monitoring

Current sweep run directory:

- `/var/log/rtlamr-sweep/20260613_143707`

Check live sweep status:

- `ssh rpi-19 "sudo cat /var/log/rtlamr-sweep/current.status"`

Check launcher output:

- `ssh rpi-19 "sudo tail -n 30 /var/log/rtlamr-sweep/launcher.log"`

Check active run log:

- `ssh rpi-19 "sudo tail -n 30 /var/log/rtlamr-sweep/20260613_143707/run.log"`

Check active run summary table:

- `ssh rpi-19 "sudo tail -n 30 /var/log/rtlamr-sweep/20260613_143707/summary.tsv"`

Find latest sweep run directory:

- `ssh rpi-19 "sudo ls -1dt /var/log/rtlamr-sweep/*/ | head -n 1"`

Interpretation:

- `current.status` should show `state=running` during sweep and `state=completed` when done.
- `summary.tsv` should accumulate one line per frequency/gain window with entries, unique IDs, and best-diff candidate.

## MAC Randomization Fix (Raspberry Pi / NetworkManager)

NetworkManager was randomizing the `wlan0` MAC address on every boot, causing the Pi to get a different IP each time.

Prior attempts that didn't work:
- `mac_addr=0` in wpa_supplicant config (wpa_supplicant handles randomization at a different layer)
- `/etc/systemd/network/10-wlan0-mac.link` with `MACAddress=...` (ineffective — systemd-networkd is not active on this Pi)

**Fix:** Create `/etc/NetworkManager/conf.d/99-no-mac-randomize.conf`:

```ini
[device]
wifi.scan-rand-mac-address=no

[connection]
wifi.cloned-mac-address=permanent
```

Then restart NetworkManager:

```bash
sudo systemctl restart NetworkManager
```

`wifi.cloned-mac-address=permanent` tells NetworkManager to use the hardware MAC address instead of generating a random one. Result: Pi gets a stable IP on every boot.
