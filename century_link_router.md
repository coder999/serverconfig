# CenturyLink/Lumen C4000BG Gateway — Notes

## Hardware

- **CPU**: Intel/Lantiq GRX500, quad-core MIPS interAptiv V2.0 ~266MHz, codename "baker"
- **RAM**: 478MB total, ~175MB free at idle
- **Flash**: 7 MTD partitions
  - mtd0: uboot (bootloader)
  - mtd1/2: ubootconfigA/B
  - mtd3: gphyfirmware
  - mtd4: dsd
  - mtd5: system_sw (~500MB, main system)
  - mtd6: res (reserved)
- **WiFi**: Intel Wave600 chipset, dual-band 2x2
  - wlan0 = 2.4GHz
  - wlan2 = 5GHz

## Software

- **OS**: OpenWrt 19.07.2 (Intel/Lantiq build), kernel 4.9.256
- **Compiled**: May 22 2025 by jenkins@baxon-nuc-11
- **SSH daemon**: Dropbear (`-T 3` = max 3 auth attempts, `-I 3600` = idle timeout)
- **Web UI**: lighttpd
- **DHCP/DNS**: dnsmasq (5 separate instances)
- **ISP management**: `greengate` process (TR-069/CWMP, phones home to CenturyLink)
- **UPnP**: both `upnpd` and `miniupnpd` running

## Network

- **WAN**: PPPoE over PTM on `ptm0.201` → `pppoe-wan3`
- **LAN**: `br-lan`, 192.168.0.0/24
- **Default gateway IP**: 192.168.0.1

## SSH Access

### Connecting

Modern OpenSSH (Windows 9.5+) fails with `kex_exchange_identification: Connection closed` — the router's Dropbear rejects the connection banner before key exchange. **Use PuTTY instead** — it negotiates the legacy algorithms Dropbear accepts without any special flags.

Negotiated cipher: AES-256-SDCTR, HMAC-SHA-256, Curve25519 key exchange.

### Credentials

- **Username**: `admin`
- **Password**: stored in `~/.ssh/router_pass.txt` (UTF-8 with BOM — strip when reading)

### Scripted access via plink

```bash
ROUTER_PASS=$(sed 's/^\xEF\xBB\xBF//' ~/.ssh/router_pass.txt | tr -d '\r\n')
"/c/Program Files/PuTTY/plink.exe" -ssh -pw "$ROUTER_PASS" -batch admin@192.168.0.1 'your command here'
```

### Troubleshooting

- **Connection reset immediately**: The SSH daemon may have a stale session from a previously closed terminal. Dropbear appears to have a low max-session limit. Fix: toggle SSH off/on in the web UI, or reboot the router.
- **Access denied with plink**: Check for special characters in the password (especially `!`) — shell history expansion can mangle it. Read from file instead of env var.
- **`router_pass.txt` has a UTF-8 BOM**: PowerShell's `Out-File -Encoding utf8` adds a BOM. Strip with `sed 's/^\xEF\xBB\xBF//'` before passing to plink.

## Shell Environment

The `admin` user is **chrooted** (group: `chroot-users`, uid 10003). This severely limits what's accessible:

- **Available binaries**: `ash`, `cat`, `echo`, `ls`, `ping`, `ps`, `date`, `grep`, `sleep`, `watch`, `mpstat` (in `/bin`); `id`, `top`, `uptime`, `free`, `nslookup`, `tail`, `less` (in `/usr/bin`)
- **No**: `find`, `wc`, `head`, `uname`, `ifconfig`, `ip`, `awk`, `sed`
- **`/proc/net`**: mostly readable — ARP table, routing table, interface stats, WiFi driver stats
- **`/proc/net/nf_conntrack`**: permission denied
- **`/var/etc`**: not visible from chroot (blocks access to dnsmasq configs and DHCP leases)
- **`/tmp`**: does not exist in the chroot
- **`/etc/dropbear`**: permission denied (can't add authorized_keys)

Key implication: **DHCP leases and device hostnames are not accessible via SSH** — use the web UI for that.

## Useful /proc Paths

| Path | Contents |
|------|----------|
| `/proc/net/arp` | LAN device MAC/IP table |
| `/proc/net/dev` | Per-interface byte/packet counters |
| `/proc/net/route` | Routing table (hex encoded) |
| `/proc/net/pppoe` | Active PPPoE session (WAN MAC) |
| `/proc/net/mtlk/wlan0/sta_list` | 2.4GHz connected WiFi clients |
| `/proc/net/mtlk/wlan2/sta_list` | 5GHz connected WiFi clients |
| `/proc/net/mtlk/wlan0/radio_cfg` | 2.4GHz radio config |
| `/proc/net/mtlk/wlan2/radio_cfg` | 5GHz radio config |
| `/proc/net/mtlk/version` | WiFi driver/firmware version |
| `/proc/mtd` | Flash partition layout |
| `/proc/mounts` | Mounted filesystems |
| `/proc/cpuinfo` | CPU details |
| `/proc/meminfo` | Memory stats |
| `/proc/uptime` | Uptime in seconds |
| `/proc/version` | Kernel version + build info |
