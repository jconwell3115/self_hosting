# OpenVPN-AS Quadlet Troubleshooting Checklist

**Summary:** Quick diagnostics and fixes for when the OpenVPN‑AS web UI (ports 943/443) is unreachable or VPN clients connect but have no Internet.

---

#### 1) Find unit & show service logs
Inspect the unit name, status and recent logs. Replace `<UNIT>` with your unit name (e.g. `openvpnas.service`).
```bash
systemctl list-units --type=service --all | grep -i openvpn
# Replace <UNIT> with the unit name from above
systemctl status <UNIT> --no-pager
journalctl -u <UNIT> -b --no-pager | tail -n 200
```

#### 2) Check listening ports & bindings
Look for ports `943` (web UI), `443` (SSL) and `1194` (OpenVPN).

```bash
ss -tlnp | egrep ':(443|943|1194)' || true
# broader
ss -tlnp | grep -i openvpn || true
```

Interpretation:
- `127.0.0.1:943` = bound to localhost only (remote browsers can't reach).
- `0.0.0.0:943` or `:::943` = listening on all interfaces (remote access possible unless firewall blocks).

#### 3) Local access tests (run on server)
Test whether the service responds locally.

```bash
curl -vk https://127.0.0.1:943/ 2>&1 | sed -n '1,120p'
curl -vk https://$(hostname -I | awk '{print $1}'):943/ 2>&1 | sed -n '1,120p'
```

If loopback works but host-IP fails → likely firewall or binding issue.

#### 4) Firewall (firewalld) checks & fixes
Check firewalld and open ports if needed.

```bash
sudo firewall-cmd --state || true
sudo firewall-cmd --list-all --zone=public
sudo firewall-cmd --list-ports
sudo firewall-cmd --permanent --list-ports
```

Open ports (if blocked):

```bash
sudo firewall-cmd --add-port=943/tcp --permanent
sudo firewall-cmd --add-port=443/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

If you need to allow masquerade for VPN client internet access via firewalld:

```bash
sudo firewall-cmd --zone=public --add-masquerade --permanent
sudo firewall-cmd --reload
```

#### 5) SELinux (if enabled)
Check SELinux mode and recent denials:

```bash
getenforce
sudo ausearch -m AVC --start today | tail -n 50 || true
# optional (requires sealert package)
sudo sealert -a /var/log/audit/audit.log 2>/dev/null | sed -n '1,120p' || true
```

AVC log entries indicate SELinux denials that may need a targeted policy.

#### 6) Inspect the quadlet unit / container invocation
Show the unit file to examine `ExecStart` and port mappings:

```bash
# Replace <UNIT> with the actual unit name
systemctl cat <UNIT>
```

What to look for:
- `podman`/`docker`/`systemd-nspawn` invocation
- `--publish` / `-p` flags or `--network host`
If no published ports and the container uses bridge networking, host access will fail.

#### 7) Fixes for binding/publish issues
- If service bound to localhost only: reconfigure OpenVPN-AS to bind `0.0.0.0`, or change the unit to publish ports.
- If container needs publishing (example `podman`):

```bash
podman run -d --name openvpnas -p 943:943 -p 443:443 ... {image}
```

- If quadlet generated the unit: edit the quadlet config or add a systemd drop‑in to include port publish or `--network host`.

After changes:

```bash
sudo systemctl daemon-reload
sudo systemctl restart <UNIT>
systemctl status <UNIT> --no-pager
ss -tlnp | egrep ':(443|943|1194)' || true
```

#### 8) Remote access test (from another machine)
Replace `SERVER` with your server IP or hostname.

```bash
curl -vk https://SERVER:943/ 2>&1 | sed -n '1,120p'
# Or open https://SERVER:943/ in a browser
```

---
---

## VPN connects but no Internet — client checks first
On the client (while connected):

```bash
ip addr show
ip route show
resolvectl status || cat /etc/resolv.conf
ping -c 4 8.8.8.8
ping -c 4 google.com
curl -s https://api.ipify.org
```

Interpretation:
- `ping 8.8.8.8` works but `ping google.com` fails → DNS issue.
- Neither works → routing/NAT/forwarding issue on server.

#### 1) Server — enable IP forwarding (if disabled)
Check and enable IPv4 forwarding:

```bash
sysctl net.ipv4.ip_forward
# If 0, enable temporarily:
sudo sysctl -w net.ipv4.ip_forward=1
# Make persistent:
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-openvpn.conf
sudo sysctl --system
```

#### 2) Server — NAT / masquerade (nft/iptables/firewalld)
Inspect NAT rules:

```bash
# nft
sudo nft list ruleset | sed -n '1,200p'
# or iptables (legacy)
sudo iptables -t nat -S || true
```

If using firewalld, enable masquerade:

```bash
sudo firewall-cmd --zone=public --add-masquerade --permanent
sudo firewall-cmd --reload
```

Example iptables MASQUERADE (replace `<VPN_SUBNET>` and `<OUT_IF>`):

```bash
sudo iptables -t nat -A POSTROUTING -s <VPN_SUBNET> -o <OUT_IF> -j MASQUERADE
```

Persist iptables/nft rules by your distro method, or prefer firewalld masquerade.

#### 3) Server/container networking notes
- If OpenVPN-AS runs in a container:
  - Host networking (`--network host`) lets host NAT/firewall apply.
  - Bridge networking requires published ports and host NAT for a VPN subnet.
- Inspect the unit and logs:

```bash
systemctl cat {{UNIT}}
journalctl -u {{UNIT}} -n 200 --no-pager
```

### 4) DNS-specific fixes
If DNS is missing or wrong, either configure OpenVPN-AS to push DNS, or temporarily force a DNS on the client:

```bash
# Example: for NetworkManager VPN connection (client)
sudo nmcli connection modify <vpn-connection-name> ipv4.dns "8.8.8.8" ipv4.ignore-auto-dns no
```

Or add `dhcp-option DNS 8.8.8.8` in OpenVPN client config.

#### 5) Connection tracking / NAT verification (advanced)
Check conntrack or NAT translations:

```bash
sudo conntrack -L | grep <CLIENT_IP> || true
sudo nft list table nat | sed -n '1,200p'
```

If no NAT translations exist for client traffic, NAT rule or forwarding isn’t applied.

#### 6) After changes — restart and re-test
```bash
sudo systemctl daemon-reload
sudo systemctl restart <UNIT>
# On client:
ip addr show
ip route show
ping -c 4 8.8.8.8
curl -s https://api.ipify.org
```

---

## Quick common fixes summary
- Enable IP forwarding on server: `net.ipv4.ip_forward = 1`.
- Add masquerade/NAT for VPN subnet to external interface (use `firewall-cmd --add-masquerade` or iptables POSTROUTING MASQUERADE).
- Ensure OpenVPN-AS pushes routes and DNS (enable “redirect gateway” if you want all traffic through VPN).
- If OpenVPN-AS runs in a container with bridge networking, switch to host networking or set up host NAT/publishing.
- Check DNS separately if IP pings work but hostnames fail.

---

## If still broken — collect these outputs and research the results with AI or Google
On client:

    ip addr show
    ip route show
    resolvectl status || cat /etc/resolv.conf
    ping -c 4 8.8.8.8

On server:

    sudo sysctl net.ipv4.ip_forward
    sudo nft list ruleset || sudo iptables -t nat -S
    sudo firewall-cmd --list-all --zone=public     # if using firewalld
    journalctl -u <UNIT> -n 200 --no-pager

Also include:
- `<UNIT>` (unit name)
- your VPN subnet (`<VPN_SUBNET>`)
- server external interface (`<OUT_IF>`)
- an example client IP

---

**Placeholders:** Replace `<UNIT>`, `<VPN_SUBNET>`, `<OUT_IF>`, and `SERVER` with your actual unit name, VPN subnet CIDR, external interface name, and server IP/hostname before running commands.
