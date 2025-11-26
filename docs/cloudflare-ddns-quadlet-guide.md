# Cloudflare DDNS Quadlet Guide

## Introduction
This guide provides a comprehensive walkthrough for running `qdm12/cloudflare-ddns` as a Podman Quadlet. It covers installation, configuration, best practices, health checks, and troubleshooting tips to ensure smooth operation.

## Prerequisites
- Podman installed on your system.
- A Cloudflare account with API access.
- Basic understanding of Linux command line.

## Installation Steps
1. **Install Podman** (if not already installed):
   ```bash
   sudo apt-get update
   sudo apt-get install podman
   ```

2. **Clone the repository**:
   ```bash
   git clone https://github.com/qdm12/cloudflare-ddns.git
   cd cloudflare-ddns
   ```

3. **Create a Quadlet configuration file**:
   ```bash
   sudo mkdir -p /etc/containers/quadlet.d
   sudo nano /etc/containers/quadlet.d/cloudflare-ddns.quadlet
   ```

   Add the following content:
   ```ini
   [Service]
   ExecStart=/usr/bin/podman run --rm \ 
       --name cloudflare-ddns \ 
       -e "API_KEY=your_api_key" \ 
       -e "ZONE=your_zone" \ 
       qdm12/cloudflare-ddns
   ```

4. **Enable the service**:
   ```bash
   sudo systemctl enable --now cloudflare-ddns.quadlet
   ```

## Configuration Guide
- Ensure you replace `your_api_key` and `your_zone` with your actual Cloudflare API key and zone.
- Adjust timeouts and other parameters as necessary based on your network conditions and preferences.

## Best Practices
- **Run on dedicated resources**: Ensure Podman runs on a machine with enough resources for reliable operation.
- **Backup configurations**: Regularly backup your Quadlet files and configurations.
- **Monitor Logs**: Use `journalctl -u cloudflare-ddns.quadlet` to monitor logs.

## Health Checks
- Check the status of the service:
   ```bash
   sudo systemctl status cloudflare-ddns.quadlet
   ```
- Verify that the DDNS service is updating correctly by checking the DNS records in your Cloudflare account.

## Troubleshooting Tips
- If the service fails to start, check the logs for errors:
   ```bash
   journalctl -xe
   ```
- Ensure your API key and zone are correctly configured in the Quadlet file.
- Validate that Podman is running without issues and troubleshoot any network configuration problems.

## Conclusion
By following this guide, you should be able to effectively run `qdm12/cloudflare-ddns` as a Podman Quadlet with best practices, health checks, and troubleshooting strategies in place. Keep this guide handy for future reference and troubleshooting!