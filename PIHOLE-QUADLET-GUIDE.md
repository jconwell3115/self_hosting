# Pi-hole with Podman Quadlets — Setup Guide

## Overview

This guide walks through running Pi-hole as a rootful Podman container managed by
systemd quadlets. Pi-hole acts as a network-wide DNS sinkhole, blocking ads and
tracking domains before they reach your devices.

## Prerequisites

- Linux (systemd-based distribution — e.g., Fedora, RHEL, CentOS Stream, Debian, Ubuntu)
- Podman 4.4+ (quadlet support built-in via `podman generate systemd` replacement)
- systemd
- Root/sudo access
- `firewalld` (or equivalent firewall)

## Directory Structure

```
/etc/containers/systemd/pihole/   # Deployed quadlet files (systemd reads from here)
  pihole.container
  pihole.network

/path/to/your/containers/pihole/  # Source files (edit these, then deploy)
  pihole.env                      # Secrets — keep permissions 600, owned by root
```

Podman named volumes (created manually before first start):

```
pihole-etc       # Pi-hole configuration, blocklists, and FTL database
pihole-dnsmasq   # dnsmasq drop-in configuration
```

---

## Quadlet Files Explained

### `pihole.network`

Defines a dedicated bridge network for Pi-hole (and optionally Unbound).

```ini
[Unit]
Description=Pi-hole Network
Documentation=https://docs.podman.io/en/latest/markdown/podman-network-create.1.html

[Network]
# Internal name used by containers: Network=pihole.network
NetworkName=pihole

# Private subnet for container-to-container communication.
# These IPs are only reachable between containers on this network —
# LAN clients connect via the host IP, not these.
Subnet=172.18.0.0/24
Gateway=172.18.0.1

# Standard Linux bridge driver
Driver=bridge
# If adding Unbound later, assign it: IP=172.18.0.3

[Install]
WantedBy=multi-user.target default.target
```

**Why a dedicated network?**
Isolates Pi-hole's traffic from other containers. Also enables container-name DNS
resolution — if you add Unbound, Pi-hole can reach it at its static IP rather than
needing host networking.

---

### `pihole.container`

```ini
[Unit]
Description=Pi-hole DNS Ad Blocker
# Don't start until the network stack is fully up
After=network-online.target
Wants=network-online.target

[Container]
ContainerName=pihole
Image=docker.io/pihole/pihole:latest

# Attach to the dedicated bridge network defined in pihole.network
Network=pihole.network
# Static IP so Unbound (or other containers) can reliably reach Pi-hole
IP=172.18.0.2

# --- Port Mappings ---
# Bind DNS (port 53) to specific host IPs rather than 0.0.0.0.
# This exposes DNS only on your LAN interface and loopback —
# not on every interface (e.g., VPN tunnels, other bridges, etc.).
#
# Replace <YOUR_SERVER_LAN_IP> with your host's LAN IP (e.g., 192.168.1.100)
PublishPort=<YOUR_SERVER_LAN_IP>:53:53/tcp
PublishPort=<YOUR_SERVER_LAN_IP>:53:53/udp
# Loopback — lets the host itself use Pi-hole for DNS
PublishPort=127.0.0.1:53:53/tcp
PublishPort=127.0.0.1:53:53/udp
# 127.0.0.53 is the address systemd-resolved listens on by default.
# Binding here lets systemd-resolved forward upstream queries to Pi-hole.
PublishPort=127.0.0.53:53:53/tcp
PublishPort=127.0.0.53:53:53/udp
# DHCP — only needed if you want Pi-hole to hand out leases (optional)
PublishPort=67:67/udp
# Web admin interface — change the host port (8082) if it conflicts
PublishPort=8082:80/tcp

# --- Volumes ---
# :Z sets the SELinux label so the rootful container can write to the volume
Volume=pihole-etc:/etc/pihole:Z
Volume=pihole-dnsmasq:/etc/dnsmasq.d:Z

# --- Secrets / Environment ---
# Keep this file owned by root with mode 600.
# Contains: WEBPASSWORD, PIHOLE_DNS_
EnvironmentFile=/path/to/your/containers/pihole/pihole.env

# Your local timezone — used for log timestamps and scheduled tasks in Pi-hole
Environment=TZ=America/New_York

# DNSMASQ_LISTENING=all tells dnsmasq to accept queries on all container interfaces.
# Needed because DNS queries arrive on the bridge interface, not eth0.
Environment=DNSMASQ_LISTENING=all

# Hostname shown in the Pi-hole web UI header
Environment=VIRTUAL_HOST=pihole.local

# --- Capabilities ---
# NET_ADMIN: required for dnsmasq to manage DHCP leases and set socket options
# NET_RAW:   required for Pi-hole to send ICMP pings (used in some DNS checks)
AddCapability=NET_ADMIN
AddCapability=NET_RAW

[Service]
Restart=always
# Pi-hole can take time on first start (gravity download). 900s = 15 minutes.
TimeoutStartSec=900
# When the container restarts, Podman rebuilds the network bridge which disrupts
# firewalld's runtime nftables masquerade rules — even if --permanent is set.
# Reloading firewalld after start re-applies all permanent rules cleanly.
ExecStartPost=/usr/bin/firewall-cmd --reload

[Install]
WantedBy=multi-user.target default.target
```

---

## Installation

### 1. Create Named Volumes

```bash
sudo podman volume create pihole-etc
sudo podman volume create pihole-dnsmasq
```

### 2. Create the Environment File

```bash
sudo install -m 600 -o root -g root /dev/null /path/to/your/containers/pihole/pihole.env
sudo nano /path/to/your/containers/pihole/pihole.env
```

Contents (no quotes, no spaces around `=`):

```bash
WEBPASSWORD=your-strong-password-here
# Semicolon-separated upstream DNS servers
PIHOLE_DNS_=1.1.1.1;8.8.8.8
```

### 3. Customize the Quadlet Files

Edit `pihole.container` and update the following before deploying:

| Setting | Description |
|---|---|
| `<YOUR_SERVER_LAN_IP>` | Your host's LAN IP address |
| `TZ=America/New_York` | Your timezone ([list](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)) |
| `VIRTUAL_HOST=pihole.local` | Hostname or IP shown in the web UI header |
| `EnvironmentFile=` path | Absolute path to your `pihole.env` file |
| `8082:80/tcp` | Change `8082` if that host port is already in use |

### 4. Deploy Quadlet Files

```bash
sudo mkdir -p /etc/containers/systemd/pihole
sudo cp pihole.container pihole.network /etc/containers/systemd/pihole/
sudo systemctl daemon-reload
```

### 5. Enable and Start

```bash
sudo systemctl enable pihole.service
sudo systemctl start pihole.service
```

Verify it started:

```bash
sudo systemctl status pihole.service
sudo podman ps
```

### 6. Configure Firewall

```bash
# Allow DNS queries from LAN clients through the firewall
sudo firewall-cmd --permanent --add-service=dns
# Enable masquerading so the container can reach upstream DNS servers
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --list-all
sudo firewall-cmd --query-masquerade
```

> **Note on masquerade and container restarts:** Even with `--permanent` set,
> masquerade rules can stop working after the container restarts. This happens
> because Podman tears down and recreates the bridge interface on each container
> start, which disrupts firewalld's runtime nftables state. The `ExecStartPost`
> line in the quadlet handles this automatically by issuing `firewall-cmd --reload`
> after every container start.

### 7. Set the Admin Password

For Pi-hole v6+, `WEBPASSWORD` in the env file only applies on first initialization.
To set or reset the password at any time:

```bash
# Interactive prompt (recommended)
sudo podman exec -it pihole pihole setpassword

# Or pass directly
sudo podman exec -it pihole pihole setpassword 'YourNewPassword'
```

### 8. Access the Web UI

Navigate to `http://<YOUR_SERVER_LAN_IP>:8082/admin`

### 9. Configure DNS on Your Network

**Router (recommended):** Set the primary DNS server in your router's DHCP settings
to your server's LAN IP. All devices receive Pi-hole automatically without
per-device configuration.

**Per-device:** Set the DNS server in each device's network settings to your
server's LAN IP.

---

## Adding Unbound (Optional)

Unbound is a recursive DNS resolver that queries root servers directly instead of
forwarding to Cloudflare/Google. This removes third-party DNS providers from your
query path entirely.

```
Without Unbound:  Device → Pi-hole → Cloudflare/Google → Internet
With Unbound:     Device → Pi-hole → Unbound → Root Servers → Internet
```

**Benefits:**
- Queries never leave your network to a commercial DNS provider
- DNSSEC validation at the resolver level
- DNS response caching

**When to skip it:**
- You trust Cloudflare/Google DNS and prefer simplicity
- Very resource-constrained systems (Unbound overhead is minimal but nonzero)

To add Unbound:

1. Create `unbound.container` in `/etc/containers/systemd/` and attach it to
   `pihole.network` with `IP=172.18.0.3`
2. Configure Unbound to listen on port `5335` and accept queries from `172.18.0.2`
3. Update `PIHOLE_DNS_=172.18.0.3#5335` in `pihole.env`
4. Restart Pi-hole: `sudo systemctl restart pihole.service`

---

## Automated Gravity Updates

Pi-hole's blocklists are managed by a database called **gravity**. Keeping it
updated ensures new ad/tracking domains are blocked promptly.

The automation uses two systemd unit files:

**`pihole-gravity-update.timer`**
- Runs every Sunday at 2:00 AM
- `Persistent=true` — catches up on missed runs after a reboot
- `RandomizedDelaySec=15m` — avoids colliding with other scheduled maintenance

**`pihole-gravity-update.service`**
- Type `oneshot` — runs once and exits (not a persistent daemon)
- Requires Pi-hole to be running before it starts
- Runs `pihole -g` inside the container
- Logs to `/var/log/pihole/gravity-update.log`
- Optionally pushes status to an uptime monitor

### Setup

```bash
sudo cp pihole-gravity-update.service pihole-gravity-update.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now pihole-gravity-update.timer
```

Edit the service file to configure the uptime monitor push URL (optional):

```ini
Environment="UPTIME_KUMA_PUSH_URL=https://your-uptime-monitor/api/push/YOUR_KEY"
```

The monitor type should be **Push** with a heartbeat interval of **10080 minutes**
(7 days).

### Verify

```bash
# Confirm timer is scheduled
sudo systemctl list-timers pihole-gravity-update.timer

# Run manually to test
sudo systemctl start pihole-gravity-update.service

# View logs
sudo tail -f /var/log/pihole/gravity-update.log
sudo journalctl -u pihole-gravity-update.service -n 50
```

---

## Troubleshooting

### DNS queries arrive but time out

The container is receiving queries but replies aren't getting back to clients.
This is almost always a missing masquerade rule:

```bash
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --reload
```

To watch for dropped packets in real time:

```bash
sudo firewall-cmd --set-log-denied=all
sudo firewall-cmd --runtime-to-permanent
sudo journalctl -k -f
```

### Confirm DNS traffic is arriving at the host

```bash
# On the server — watch all DNS traffic
sudo tcpdump -i any port 53 -n

# From a LAN client
dig @<YOUR_SERVER_LAN_IP> google.com
```

### Port 53 already in use

```bash
sudo ss -tulpn | grep :53
```

If `systemd-resolved` is occupying port 53, disable its stub listener:

```bash
sudo sed -i 's/#DNSStubListener=yes/DNSStubListener=no/' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
```

### Pi-hole web UI login fails (v6+)

`WEBPASSWORD` in the env file is only read on first initialization. Reset with:

```bash
sudo podman exec -it pihole pihole setpassword
```

### Masquerade stops working after container restart

The `ExecStartPost=/usr/bin/firewall-cmd --reload` line in the quadlet handles this
automatically. If it keeps breaking, verify the line is present in the deployed file:

```bash
sudo grep ExecStartPost /etc/containers/systemd/pihole/pihole.container
```

If missing, redeploy:

```bash
sudo cp pihole.container /etc/containers/systemd/pihole/pihole.container
sudo systemctl daemon-reload
sudo systemctl restart pihole.service
```

### Check what is listening on DNS/web ports

```bash
sudo ss -tulpn | grep -E ":(53|67|8082)\b"
```

---

## Backup and Restore

### Backup

```bash
sudo podman volume export pihole-etc > pihole-etc-$(date +%Y%m%d).tar
sudo podman volume export pihole-dnsmasq > pihole-dnsmasq-$(date +%Y%m%d).tar
```

### Restore

```bash
cat pihole-etc-YYYYMMDD.tar | sudo podman volume import pihole-etc -
cat pihole-dnsmasq-YYYYMMDD.tar | sudo podman volume import pihole-dnsmasq -
sudo systemctl restart pihole.service
```

---

## Useful Commands

```bash
# Service management
sudo systemctl status pihole.service
sudo systemctl restart pihole.service
sudo podman logs pihole

# Pi-hole CLI
sudo podman exec -it pihole pihole status
sudo podman exec -it pihole pihole restartdns
sudo podman exec pihole pihole -g            # manual gravity update
sudo podman exec -it pihole pihole -t        # live query log tail

# Update the container image
sudo podman pull docker.io/pihole/pihole:latest
sudo systemctl restart pihole.service

# Gravity update timer
sudo systemctl status pihole-gravity-update.timer
sudo systemctl list-timers pihole-gravity-update.timer
sudo tail -f /var/log/pihole/gravity-update.log
```

---

## Recommended Blocklists

Beyond the default StevenBlack list, consider:

- **HaGeZi Pro** — broad coverage, low false positives:
  `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/domains/pro.txt`
- **HaGeZi TIF** (Threat Intelligence Feeds) — malware/phishing domains:
  `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/domains/tif.txt`
- **Firebog** — curated collection of lists by category: https://firebog.net/

Add lists via the Pi-hole web UI under **Adlists**, then run a gravity update.

---

## Security Notes

- Keep `pihole.env` owned by root with mode `600`
- Set a strong admin password immediately after setup
- Bind DNS ports to specific interfaces, not `0.0.0.0`
- Keep the container image updated regularly
- Enable DNSSEC in Pi-hole settings for additional validation

---

## References

- [Pi-hole Documentation](https://docs.pi-hole.net/)
- [Pi-hole Docker Image](https://github.com/pi-hole/docker-pi-hole)
- [Podman Quadlet Documentation](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
- [Unbound with Pi-hole](https://docs.pi-hole.net/guides/dns/unbound/)
- [Unbound Project](https://nlnetlabs.nl/projects/unbound/about/)
