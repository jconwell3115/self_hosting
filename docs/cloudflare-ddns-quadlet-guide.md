# Complete How-To: Running qdm12/cloudflare-ddns as a Podman Quadlet

This comprehensive guide will walk you through setting up qdm12/cloudflare-ddns as a systemd-managed Podman container using Quadlet, following best practices for security, reliability, and maintainability.

## Overview

**What you'll build:**
- A systemd-managed Podman container running qdm12/cloudflare-ddns
- Secure API token handling via Podman secrets
- Automatic restart on failure with proper health checks
- Full logging to journald
- Clean boot-time startup with network dependency handling

**Prerequisites:**
- Podman installed (version 4.4+ for full Quadlet support)
- systemd-based Linux distribution
- Cloudflare account with a zone/domain
- Root or sudo access

---

## Step 1: Create Cloudflare API Token

### Generate a scoped API token (recommended over global API key)

1. Log in to Cloudflare Dashboard
2. Go to **My Profile** → **API Tokens** → **Create Token**
3.  Use the **Edit zone DNS** template or create custom token with:
   - **Permissions:** Zone → DNS → Edit
   - **Zone Resources:** Include → Specific zone → `yourdomain.com`
4. Create the token and **copy it immediately** (you won't see it again)

**Security note:** Never use your Global API Key — always use scoped tokens with minimal permissions.

---

## Step 2: Prepare Host Directories and Secrets

### Create directory structure

```bash
# Create config directory
sudo mkdir -p /etc/cloudflare-ddns
sudo chmod 700 /etc/cloudflare-ddns
```

### Store the API token securely

```bash
# Create token file (replace YOUR_TOKEN_HERE with your actual token)
printf '%s' "YOUR_TOKEN_HERE" | sudo tee /etc/cloudflare-ddns/cf_token >/dev/null
sudo chmod 600 /etc/cloudflare-ddns/cf_token
sudo chown root:root /etc/cloudflare-ddns/cf_token
```

### Create a Podman secret

```bash
# Import the token as a Podman secret
sudo podman secret create cloudflare_ddns_token /etc/cloudflare-ddns/cf_token

# Verify the secret was created
sudo podman secret ls
```

**Why use secrets? ** Podman secrets are mounted as read-only tmpfs inside containers and are never written to disk in the container filesystem, reducing exposure. 

---

## Step 3: Determine Your Configuration

### Common environment variables for qdm12/cloudflare-ddns

Based on typical usage patterns (verify against the current image documentation):

| Variable | Description | Example |
|----------|-------------|---------|
| `ZONE` or `DOMAIN` | Your Cloudflare zone/domain | `example.com` |
| `SUBDOMAIN` or `RECORD` | DNS record to update (@ for root) | `home` or `@` |
| `PROXIED` | Cloudflare proxy status | `false` (true/false) |
| `TTL` | DNS TTL in seconds | `1` (1 = automatic) |
| `INTERVAL` or `PERIOD` | Update check interval | `5m` or `300s` |
| `CF_API_TOKEN` | API token (prefer file-based) | Read from secret |
| `LOG_LEVEL` | Logging verbosity | `info` (debug/info/warning/error) |

**Note:** The qdm12 image typically supports reading configuration from environment variables. Check the image's documentation for the exact variable names as they may vary between versions.

---

## Step 4: Create the Quadlet Container Unit

### Create the systemd container unit file

```ini
# /etc/containers/systemd/cloudflare-ddns.container

[Unit]
Description=Cloudflare DDNS updater (qdm12)
Documentation=https://github.com/qdm12/cloudflare-ddns
Wants=network-online.target
After=network-online.target

[Container]
# Image
Image=docker.io/qmcgaw/cloudflare-ddns:latest
Pull=newer

# Container name
ContainerName=cloudflare-ddns

# Auto-update (requires podman-auto-update. timer)
AutoUpdate=registry

# Environment variables
# Adjust these to match your domain and preferences
Environment=ZONE=example. com
Environment=SUBDOMAIN=home
Environment=PROXIED=false
Environment=TTL=1
Environment=PERIOD=5m
Environment=LOG_LEVEL=info
Environment=HEALTH_SERVER_ADDRESS=127.0.0.1:9999

# Secret mount for API token
# The image should read CF_API_TOKEN from env, but we'll inject it via secret
Secret=cloudflare_ddns_token,type=env,target=CF_API_TOKEN

# Health check (runs inside container)
HealthCmd=/bin/sh -c "wget -q --spider http://127.0.0. 1:9999/healthz || exit 1"
HealthInterval=30s
HealthTimeout=10s
HealthRetries=3
HealthStartPeriod=10s

# Restart policy
Restart=on-failure
RestartSec=10s
RestartMaxAttempts=5

# Security options
ReadOnly=true
NoNewPrivileges=true
SecurityLabelDisable=true

# Resource limits (optional but recommended)
Memory=128M
MemorySwap=128M

# Logging
LogDriver=journald

[Service]
# Allow systemd to manage the service
TimeoutStartSec=60
Restart=on-failure

[Install]
WantedBy=multi-user.target default.target
```

**Important path note:** Quadlet units should be placed in:
- System-wide: `/etc/containers/systemd/` (recommended for root services)
- User-specific: `~/.config/containers/systemd/`

### Alternative configuration if the image doesn't support health server

If the qdm12 image doesn't expose a health endpoint, use this simpler approach:

```ini
# /etc/containers/systemd/cloudflare-ddns.container

[Unit]
Description=Cloudflare DDNS updater (qdm12)
Documentation=https://github.com/qdm12/cloudflare-ddns
Wants=network-online.target
After=network-online.target

[Container]
# Image
Image=docker.io/qmcgaw/cloudflare-ddns:latest
Pull=newer

# Container name
ContainerName=cloudflare-ddns

# Auto-update
AutoUpdate=registry

# Environment variables - CUSTOMIZE THESE
Environment=ZONE=example.com
Environment=SUBDOMAIN=home
Environment=PROXIED=false
Environment=TTL=1
Environment=PERIOD=5m
Environment=LOG_LEVEL=info

# Secret mount for API token
Secret=cloudflare_ddns_token,type=env,target=CF_API_TOKEN

# Restart policy
Restart=always
RestartSec=30s

# Security options
ReadOnly=true
NoNewPrivileges=true

# Resource limits
Memory=128M

# Logging
LogDriver=journald

[Service]
TimeoutStartSec=60
Restart=always

[Install]
WantedBy=multi-user.target default.target
```

---

## Step 5: Enable and Start the Service

### Reload systemd and start

```bash
# Reload systemd to discover the new Quadlet unit
sudo systemctl daemon-reload

# Enable the service (start at boot)
sudo systemctl enable cloudflare-ddns. service

# Start the service now
sudo systemctl start cloudflare-ddns.service

# Check status
sudo systemctl status cloudflare-ddns.service
```

**Note:** Quadlet automatically converts `. container` files to systemd service units. The service will be named `cloudflare-ddns. service`. 

---

## Step 6: Verify Operation

### Check service status

```bash
# Full status
sudo systemctl status cloudflare-ddns.service

# Should show "active (running)"
```

### View logs

```bash
# Follow logs in real-time
sudo journalctl -u cloudflare-ddns.service -f

# View recent logs
sudo journalctl -u cloudflare-ddns.service -n 50

# Logs since last boot
sudo journalctl -u cloudflare-ddns. service -b
```

### Check container status

```bash
# List running containers
sudo podman ps | grep cloudflare-ddns

# Inspect the container
sudo podman inspect cloudflare-ddns

# Check health (if healthcheck is configured)
sudo podman healthcheck run cloudflare-ddns
```

### Verify DNS update

```bash
# Query your DNS record
dig +short home.example.com

# Or use nslookup
nslookup home.example.com

# Check in Cloudflare Dashboard
# Go to DNS → Records and verify the A/AAAA record shows your current IP
```

---

## Step 7: Enable Auto-Updates (Optional but Recommended)

### Enable Podman auto-update timer

```bash
# Enable the auto-update timer (checks for new images daily)
sudo systemctl enable --now podman-auto-update. timer

# Check timer status
sudo systemctl status podman-auto-update.timer

# Manually trigger an update check
sudo podman auto-update

# View auto-update logs
sudo journalctl -u podman-auto-update.service
```

The `AutoUpdate=registry` line in the Quadlet unit tells Podman to pull newer images and restart the container when updates are available.

---

## Troubleshooting Guide

### Problem: Service fails to start

**Symptoms:**
```
● cloudflare-ddns.service - Cloudflare DDNS updater (qdm12)
     Loaded: loaded
     Active: failed (Result: exit-code)
```

**Solutions:**

1. **Check logs for error messages:**
   ```bash
   sudo journalctl -u cloudflare-ddns.service -n 100 --no-pager
   ```

2. **Common causes:**
   - **Invalid API token:** Verify token in `/etc/cloudflare-ddns/cf_token`
   - **Wrong zone/subdomain:** Check `ZONE` and `SUBDOMAIN` match your Cloudflare setup
   - **Network not ready:** Ensure `network-online.target` is enabled:
     ```bash
     sudo systemctl enable systemd-networkd-wait-online.service
     ```
   - **Secret not found:** Verify secret exists:
     ```bash
     sudo podman secret ls
     ```

3. **Test the container manually:**
   ```bash
   # Run interactively to see errors
   sudo podman run --rm -it \
     --secret cloudflare_ddns_token,type=env,target=CF_API_TOKEN \
     -e ZONE=example.com \
     -e SUBDOMAIN=home \
     -e LOG_LEVEL=debug \
     docker.io/qmcgaw/cloudflare-ddns:latest
   ```

---

### Problem: DNS record not updating

**Symptoms:**
- Service running but DNS record shows old IP

**Solutions:**

1. **Increase log verbosity:**
   ```bash
   # Edit the unit file
   sudo nano /etc/containers/systemd/cloudflare-ddns.container
   # Change: Environment=LOG_LEVEL=debug
   
   sudo systemctl daemon-reload
   sudo systemctl restart cloudflare-ddns.service
   sudo journalctl -u cloudflare-ddns.service -f
   ```

2. **Verify API token permissions:**
   - Log in to Cloudflare
   - Check token has Zone → DNS → Edit for the correct zone
   - Regenerate token if needed and update secret:
     ```bash
     printf '%s' "NEW_TOKEN" | sudo tee /etc/cloudflare-ddns/cf_token >/dev/null
     sudo podman secret rm cloudflare_ddns_token
     sudo podman secret create cloudflare_ddns_token /etc/cloudflare-ddns/cf_token
     sudo systemctl restart cloudflare-ddns.service
     ```

3. **Check IP detection:**
   - The container needs to detect your public IP
   - If behind NAT/firewall, ensure outbound HTTP/HTTPS is allowed
   - Check logs for IP detection errors

4. **Verify Cloudflare API access:**
   ```bash
   # Test API manually (replace TOKEN and ZONE_ID)
   curl -X GET "https://api.cloudflare. com/client/v4/zones/ZONE_ID/dns_records" \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Content-Type: application/json"
   ```

---

### Problem: Container keeps restarting

**Symptoms:**
```
sudo podman ps -a
# Shows container repeatedly restarting
```

**Solutions:**

1. **Check restart count and reason:**
   ```bash
   sudo podman inspect cloudflare-ddns | grep -A 5 "State"
   sudo journalctl -u cloudflare-ddns.service | grep -i error
   ```

2. **Common causes:**
   - **Missing required env vars:** Ensure ZONE and SUBDOMAIN are set
   - **Invalid configuration:** Check all environment variables for typos
   - **Health check failing:** If using HealthCmd, verify the endpoint works

3. **Disable restart temporarily for debugging:**
   ```bash
   # Edit unit file, change Restart=on-failure to Restart=no
   sudo systemctl daemon-reload
   sudo systemctl restart cloudflare-ddns.service
   # Check logs without auto-restart interference
   ```

---

### Problem: Service not starting at boot

**Symptoms:**
- Service works when started manually but doesn't start after reboot

**Solutions:**

1. **Verify service is enabled:**
   ```bash
   sudo systemctl is-enabled cloudflare-ddns.service
   # Should show "enabled"
   ```

2. **Enable if not enabled:**
   ```bash
   sudo systemctl enable cloudflare-ddns.service
   ```

3. **Check dependencies:**
   ```bash
   # Ensure network-online.target is reached
   sudo systemctl status network-online.target
   
   # Enable network wait service if needed
   sudo systemctl enable systemd-networkd-wait-online.service
   ```

4. **Check boot logs:**
   ```bash
   sudo journalctl -u cloudflare-ddns.service -b
   ```

---

### Problem: Can't view logs or service not found

**Symptoms:**
```
Failed to start cloudflare-ddns.service: Unit cloudflare-ddns.service not found. 
```

**Solutions:**

1.  **Verify Quadlet file location:**
   ```bash
   ls -la /etc/containers/systemd/cloudflare-ddns.container
   ```

2. **Reload systemd daemon:**
   ```bash
   sudo systemctl daemon-reload
   ```

3. **Check for syntax errors:**
   ```bash
   # Quadlet should generate the unit; check for errors
   sudo /usr/libexec/podman/quadlet --dryrun
   # Or on some systems:
   sudo /usr/lib/podman/quadlet --dryrun
   ```

4.  **Verify Podman Quadlet support:**
   ```bash
   podman --version
   # Should be 4.4. 0 or higher
   
   # Check if quadlet generator exists
   ls -la /usr/lib/systemd/system-generators/*quadlet*
   ```

---

### Problem: Memory or resource issues

**Symptoms:**
- Container killed due to OOM (Out of Memory)
- High CPU usage

**Solutions:**

1. **Check resource usage:**
   ```bash
   sudo podman stats cloudflare-ddns
   ```

2. **Increase memory limit in unit file:**
   ```ini
   Memory=256M
   MemorySwap=256M
   ```

3. **Reduce update frequency:**
   ```ini
   Environment=PERIOD=10m
   ```

---

### Problem: IPv6 not updating (if dual-stack)

**Symptoms:**
- IPv4 (A record) updates correctly
- IPv6 (AAAA record) not updating

**Solutions:**

1.  **Check image documentation** for IPv6 support and configuration
2. **Verify your host has IPv6:**
   ```bash
   ip -6 addr show
   curl -6 https://ifconfig.co
   ```

3. **May need additional environment variables** for IPv6 support (check qdm12 docs)

---

## Advanced Configuration

### Running in a Podman Pod with cloudflared

If you want to run this alongside a cloudflared tunnel in the same pod (shared network namespace):

```ini
# /etc/containers/systemd/web-pod.pod

[Unit]
Description=Web services pod (cloudflared + ddns)
Wants=network-online.target
After=network-online.target

[Pod]
PodName=web-pod
PublishPort=8080:8080

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/containers/systemd/cloudflare-ddns-pod.container

[Unit]
Description=Cloudflare DDNS in web-pod
Requires=web-pod.service
After=web-pod.service

[Container]
Image=docker.io/qmcgaw/cloudflare-ddns:latest
Pod=web-pod. pod
Secret=cloudflare_ddns_token,type=env,target=CF_API_TOKEN
Environment=ZONE=example.com
Environment=SUBDOMAIN=home
Environment=PERIOD=5m

[Install]
WantedBy=multi-user.target
```

Enable both:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now web-pod.service cloudflare-ddns-pod. service
```

---

## Security Best Practices Checklist

- ✅ Use API tokens (not Global API Key)
- ✅ Scope token to minimum permissions (Zone DNS Edit only)
- ✅ Store token in Podman secret (not plaintext in unit file)
- ✅ Set file permissions to 600 on token file
- ✅ Use `ReadOnly=true` for container filesystem
- ✅ Use `NoNewPrivileges=true` to prevent privilege escalation
- ✅ Set memory limits to prevent resource exhaustion
- ✅ Use `LogDriver=journald` for centralized logging
- ✅ Enable auto-updates to get security patches
- ✅ Run as non-root user in container (qdm12 image should do this by default)

---

## Maintenance Tasks

### Update the image manually

```bash
# Pull latest image
sudo podman pull docker.io/qmcgaw/cloudflare-ddns:latest

# Restart service (Quadlet will use new image)
sudo systemctl restart cloudflare-ddns.service
```

### Rotate API token

```bash
# Create new token in Cloudflare Dashboard
# Update secret
printf '%s' "NEW_TOKEN" | sudo tee /etc/cloudflare-ddns/cf_token >/dev/null
sudo podman secret rm cloudflare_ddns_token
sudo podman secret create cloudflare_ddns_token /etc/cloudflare-ddns/cf_token

# Restart service
sudo systemctl restart cloudflare-ddns. service
```

### Backup configuration

```bash
# Backup the Quadlet unit and token
sudo tar czf cloudflare-ddns-backup.tar.gz \
  /etc/containers/systemd/cloudflare-ddns.container \
  /etc/cloudflare-ddns/
```

### Remove/uninstall

```bash
# Stop and disable service
sudo systemctl stop cloudflare-ddns.service
sudo systemctl disable cloudflare-ddns.service

# Remove Quadlet unit
sudo rm /etc/containers/systemd/cloudflare-ddns.container

# Remove secret
sudo podman secret rm cloudflare_ddns_token

# Remove config directory
sudo rm -rf /etc/cloudflare-ddns/

# Remove container and image
sudo podman rm -f cloudflare-ddns
sudo podman rmi docker.io/qmcgaw/cloudflare-ddns:latest

# Reload systemd
sudo systemctl daemon-reload
```

---

## Quick Reference Commands

```bash
# View status
sudo systemctl status cloudflare-ddns.service

# View logs (live)
sudo journalctl -u cloudflare-ddns.service -f

# Restart service
sudo systemctl restart cloudflare-ddns.service

# Stop service
sudo systemctl stop cloudflare-ddns.service

# Start service
sudo systemctl start cloudflare-ddns.service

# Disable service (don't start at boot)
sudo systemctl disable cloudflare-ddns.service

# Enable service (start at boot)
sudo systemctl enable cloudflare-ddns.service

# Check container stats
sudo podman stats cloudflare-ddns

# Execute command in running container
sudo podman exec -it cloudflare-ddns /bin/sh

# View environment variables
sudo podman inspect cloudflare-ddns | grep -A 20 "Env"

# Test auto-update
sudo podman auto-update --dry-run
```

---

## Summary

You now have a production-ready Cloudflare DDNS service running as a systemd-managed Podman container with:

- ✅ Automatic startup at boot with proper network dependencies
- ✅ Secure token handling via Podman secrets
- ✅ Health checks and automatic restart on failure
- ✅ Centralized logging to journald
- ✅ Automatic image updates
- ✅ Resource limits and security hardening
- ✅ Easy maintenance and troubleshooting

The service will now keep your DNS records updated automatically, restart on failures, survive reboots, and update itself when new versions are available. 

**Next steps:**
- Monitor the service for the first 24-48 hours to ensure stable operation
- Set up alerting (optional) if the service fails
- Consider adding this configuration to your infrastructure-as-code or dotfiles repo

If you encounter any issues not covered in the troubleshooting section, check the qdm12/cloudflare-ddns GitHub repository for updated documentation and known issues. 
