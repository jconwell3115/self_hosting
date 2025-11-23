# How to Setup OpenVPN Access Server on Fedora Budgie Atomic Using Podman Quadlet

**Published: November 20, 2025**  
**Author: jconwell3115**

------------------------------------------------------------

## Introduction

Setting up a VPN server on an old computer can breathe new life into outdated hardware while providing secure remote access to your home network. In this comprehensive guide, we'll walk through installing and configuring OpenVPN Access Server on Fedora Budgie Atomic using Podman Quadlet - a modern, containerized approach that's perfect for immutable Linux distributions.

**What You'll Need:**
- An old computer running Fedora Budgie Atomic (or similar atomic/immutable Fedora variant)
- Basic command line knowledge
- Root/sudo access
- A stable internet connection

**Why OpenVPN Access Server?**
OpenVPN Access Server (OpenVPN-AS) is a full-featured VPN solution that provides:
- Web-based administration interface
- Easy client configuration
- Support for multiple platforms (Windows, macOS, Linux, iOS, Android)
- Enterprise-grade security
- User management and access control

------------------------------------------------------------

## Part 1: Server Setup with Podman Quadlet

### What is Podman Quadlet?

Podman Quadlet is a systemd-native way to manage containers on Linux. Unlike Docker Compose, it integrates directly with systemd, making it ideal for immutable distributions like Fedora Atomic where traditional package management is limited.

### Step 1: Create the Quadlet Configuration File

First, we'll create the systemd Quadlet file that defines our OpenVPN Access Server container:

```bash
cat << 'EOF' | sudo tee /etc/containers/systemd/openvpn-as.container
# /etc/containers/systemd/openvpn-as.container
[Unit]
Description=OpenVPN Access Server (Podman Quadlet)
Wants=network-online.target
After=network-online.target

[Container]
ContainerName=openvpn-as
# Choose your image. linuxserver/openvpn-as is popular and maintained.
# Image=lscr.io/linuxserver/openvpn-as:latest
# If you prefer the official OpenVPN Inc. image instead, use:
Image=openvpn/openvpn-as:latest

# Rootful + host net to avoid slirp quirks and preserve routing semantics.
Network=host

# Required for VPN routing + iptables operations inside the container
AddCapability=NET_ADMIN
AddCapability=NET_RAW

# Ensure the TUN device is available in the container
AddDevice=/dev/net/tun

# Persist the AS configuration and PKI
Volume=/home/rhlabs/podman/volumes/openvpn-config:/openvpn:z

# (Optional) auto-update at reboot if you want
# AutoUpdate=registry

# Environment (adjust PUID/PGID/UMASK as you like; leave root if you want fully rootful)
# If you prefer running as root inside the container, omit PUID/PGID
Environment=PUID=0
Environment=PGID=0
Environment=TZ=America/New_York

# If you need to accept EULA non-interactively with the linuxserver image:
# Environment=INTERACTIVE=false
# Environment=EULA=accept


# Open Ports
PublishPort=943:943
PublishPort=9443:9443
PublishPort=1194:1194/udp

# (Optional) if your host uses nftables backbone, you can allow legacy iptables in the container:
# Environment=OVPN_AS_USE_LEGACY_IPTABLES=1

[Service]
# Ensure the tun module exists before the container starts
ExecStartPre=/usr/sbin/modprobe tun

# Make it resilient
Restart=always
RestartSec=5s
TimeoutStartSec=0

# If your host needs IP forwarding at container start, you can uncomment:
ExecStartPre=/usr/bin/sysctl -w net.ipv4.ip_forward=1
# ExecStartPre=/usr/bin/sysctl -w net.ipv6.conf.all.forwarding=1

[Install]
WantedBy=multi-user.target
EOF
```

**Understanding the Configuration:**

- **Network=host**: Uses host networking instead of container NAT, essential for VPN routing
- **AddCapability=NET_ADMIN/NET_RAW**: Grants network manipulation permissions
- **AddDevice=/dev/net/tun**: Provides access to the TUN device for VPN tunneling
- **Volume**: Persists configuration across container restarts
- **Ports**:
  - 943: Admin and client web interface (HTTPS)
  - 9443: Web services
  - 1194/udp: Default OpenVPN data channel

### Step 2: Prepare the Host System

Create the necessary directories and enable IP forwarding:
> Change the directory structure to match yours

```bash
# Create persistent storage directory
sudo mkdir -p /home/{username}/podman/volumes/openvpn-config
sudo chown -R root:root /home/{username}/podman/volumes/openvpn-config

# Enable IP forwarding (required for VPN routing)
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-openvpn.conf
echo "net.ipv6.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.d/99-openvpn.conf
sudo sysctl --system
```

### Step 3: Configure Firewall

Open the required ports in your firewall:

```bash
# For firewalld (default on Fedora)
sudo firewall-cmd --add-port=943/tcp --add-port=9443/tcp --add-port=1194/udp --permanent
sudo firewall-cmd --reload

# Verify rules
sudo firewall-cmd --list-ports
```

### Step 4: Start the OpenVPN Service

```bash
# Reload systemd to recognize the new Quadlet
sudo systemctl daemon-reload

# Enable the service to start at boot
sudo systemctl enable openvpn-as.service

# Start the service now
sudo systemctl start openvpn-as.service

# Check service status
systemctl status openvpn-as.service

# Monitor logs (Ctrl+C to exit)
journalctl -u openvpn-as.service -f
```

### Step 5: Initial Access and Configuration

After the service starts successfully:

1. **Access the Admin Interface**: Navigate to `https://your-server-ip:943/admin`
2. **Accept the self-signed certificate** (you can configure a proper SSL certificate later)
3. **Default credentials**: 
   - Username: `openvpn`
   - Password: Check the container logs for the auto-generated password:
```bash
journalctl -u openvpn-as.service | grep -i "Auto-generated pass ="
```

4. **Complete the initial setup wizard** in the web interface

------------------------------------------------------------

## Part 2: Server Configuration and Management

### Accessing the Admin Interface

Once your server is running, you have two main interfaces:

- **Admin UI**: `https://your-server-ip:943/admin` - For server configuration and user management
- **Client UI**: `https://your-server-ip:943/` - Where users download client profiles

### Essential Configuration Steps

#### 1. Change the Admin Password

From the Admin UI:
1. Navigate to **Users** in the left menu
2. Select the `openvpn` user
3. Click **Change Password**
4. Set a strong, unique password

#### 2. Configure Network Settings

Navigate to **VPN Server -> Network Settings**:

- **Hostname or IP Address**: Set this to your public IP or domain name
- **VPN Server Port**: Default is 1194/UDP (change if needed)
- **Protocol**: UDP is recommended for better performance, TCP has better security

#### 3. Set Up Routing

Navigate to **Access Controls > Internet Access and DNS**:

- Internet Gateway:
  - Enable "Full-Tunnel"
	  - This will send all client traffic through the VPN

- **DNS Settings**:
  - Specify DNS servers to push to clients
  - Recommended: Use your local DNS or public DNS like `1.1.1.1` or `8.8.8.8`

#### 4. Create User Accounts

Navigate to **Users**:

1. Click Add New User
2. Enter username
3. Set permissions:
   - **User role**: select user for normal account
   - **Allow Auto-login**: Convenient but less secure
   - **Authentication**: Set Password
   - **Networking**: Leave these as default for ease of use
	   - Enable VPN gateway if you're using another router that services clients to connect to the VPN
1. Set password or use certificate authentication

### Advanced Configuration Options

#### SSL Certificate Configuration

For production use, replace the self-signed certificate:

1. Obtain a certificate from Let's Encrypt or your certificate authority
2. Navigate to **Certificate Management**
3. Select `use your own certificate`
4. Upload your certificate

#### Two-Factor Authentication

Enable 2FA for additional security:

1. Navigate to **Authentication > General**
2. Enable **Multifactor Authentication (MFA)**
3. Users can set up MFA using Google Authenticator or similar apps

------------------------------------------------------------

## Part 3: Client Setup and Connection

### Overview

OpenVPN Access Server supports multiple client platforms with various connection methods:

- **User-locked profiles**: Pre-configured for specific users
- **Auto-login profiles**: Embedded credentials (convenient but less secure)
- **OpenVPN Connect client**: Official client application
- **Community OpenVPN client**: Open-source alternative

### Windows Client Setup

#### Method 1: Using OpenVPN Connect (Recommended)
> **These setup methods assume that you have local access to the server and are not remote**
1. **Download OpenVPN Connect**:
   - Visit `https://openvpn.net/client/`
   - Download and install OpenVPN Connect for Windows

2. **Get Your Profile**:
   - Navigate to `https://your-server-ip:943/`
   - Log in with your username and password
   - Click **Download for Windows** or **Get Profile**

1. **(Optional) Import Profile**:
   - Open OpenVPN Connect
   - Click **Import Profile > FILE**
   - Browse to your downloaded `.ovpn` file
   - Click **Add**

4. **Connect**:
   - Toggle the connection switch
   - Enter password if using user-locked profile

#### Method 2: Using Community OpenVPN Client

1. Download from `https://openvpn.net/community-downloads/`
2. Install the client
3. Download your `.ovpn` profile from the client UI
4. Copy the profile to `C:\Program Files\OpenVPN\config\`
5. Right-click the OpenVPN GUI system tray icon
6. Select your profile and click **Connect**

### macOS Client Setup

1. **Download OpenVPN Connect**:
   - Visit `https://openvpn.net/client/`
   - Download for macOS

2. **Install and Import Profile**:
   - Open the downloaded DMG
   - Drag OpenVPN Connect to Applications
   - Launch the application
   - Click **Import Profile > FILE**
   - Select your downloaded `.ovpn` file

3. **Connect**:
   - Toggle the connection
   - Enter credentials if prompted

### Linux Client Setup

#### Using OpenVPN Connect

```bash
# Download the OpenVPN Connect client
# Visit https://openvpn.net/client/ for the latest version

# For Fedora/RHEL:
sudo dnf install openvpn

# Download your profile from the web interface
# Then connect using:
sudo openvpn --config /path/to/your-profile.ovpn
```

#### Using NetworkManager (Desktop)

```bash
# Install NetworkManager OpenVPN plugin
sudo dnf install NetworkManager-openvpn-gnome

# Import profile:
# 1. Open Settings > Network
# 2. Click '+' to add VPN
# 3. Select 'Import from file'
# 4. Browse to your .ovpn file
# 5. Click 'Add'
```

### iOS Client Setup

1. **Download OpenVPN Connect** from the App Store
2. **Transfer Profile**:
   - **Method 1**: Email the profile to yourself and open on iOS
   - **Method 2**: Navigate to client UI in Safari, download directly
   - **Method 3**: Use AirDrop from a Mac

3. **Import and Connect**:
   - Open the profile file
   - Select **OpenVPN**
   - Profile imports automatically
   - Tap **Add** then **Connect**

### Android Client Setup

1. **Download OpenVPN Connect** from Google Play Store
2. **Get Profile**:
   - Navigate to client UI in browser
   - Download profile
   - Or scan QR code if available

3. **Import and Connect**:
   - Open OpenVPN Connect
   - Tap **+** or **Import Profile**
   - Select **FILE** and browse to profile
   - Tap **Add** then **Connect**

### Troubleshooting Client Connections

#### Connection Fails

1. **Check server accessibility**:
   ```bash
   # From client machine
   ping your-server-ip
   telnet your-server-ip 1194
   ```

2. **Verify firewall rules** on server
3. **Check server logs**:
   ```bash
   journalctl -u openvpn-as.service -f
   ```

#### DNS Not Working

1. **Check DNS settings** in Admin UI > VPN Settings
2. **Verify DNS push** in client logs
3. **Test DNS manually** while connected:
   ```bash
   nslookup google.com
   ```

#### Slow Connection

1. **Try TCP instead of UDP** (Configuration > Network Settings)
2. **Reduce encryption level** if appropriate for your use case
3. **Check server resources** (CPU, bandwidth)

------------------------------------------------------------

## Part 4: Maintenance and Troubleshooting

### Monitoring the Service

```bash
# Check service status
systemctl status openvpn-as.service

# View real-time logs
journalctl -u openvpn-as.service -f

# Check last 100 log lines
journalctl -u openvpn-as.service -n 100

# View container details
sudo podman ps
sudo podman inspect openvpn-as
```

### Updating OpenVPN Access Server

```bash
# Pull latest image
sudo podman pull openvpn/openvpn-as:latest

# Restart service (Quadlet will use new image)
sudo systemctl restart openvpn-as.service
```

### Backup and Restore

#### Backup Configuration

```bash
# Stop the service
sudo systemctl stop openvpn-as.service

# Backup configuration directory
sudo tar -czf openvpn-backup-$(date +%Y%m%d).tar.gz \
  /home/rhlabs/podman/volumes/openvpn-config/

# Restart service
sudo systemctl start openvpn-as.service
```

#### Restore Configuration

```bash
sudo systemctl stop openvpn-as.service
sudo tar -xzf openvpn-backup-YYYYMMDD.tar.gz -C /
sudo systemctl start openvpn-as.service
```

### Common Issues and Solutions

#### 1. TUN Device Not Available

**Symptom**: Service fails to start, logs show TUN device errors

**Solution**:
```bash
# Load TUN module
sudo modprobe tun

# Verify it's loaded
lsmod | grep tun

# Make it persistent
echo "tun" | sudo tee /etc/modules-load.d/tun.conf
```

#### 2. IP Forwarding Not Working

**Symptom**: Clients connect but can't route traffic

**Solution**:
```bash
# Check current status
sysctl net.ipv4.ip_forward

# Should return 1, if not:
sudo sysctl -w net.ipv4.ip_forward=1

# Verify persistent configuration
cat /etc/sysctl.d/99-openvpn.conf
```

#### 3. Port Already in Use

**Symptom**: Service fails to bind to ports

**Solution**:
```bash
# Check what's using the port
sudo ss -tulpn | grep :943
sudo ss -tulpn | grep :1194

# Stop conflicting service or change OpenVPN ports
```

#### 4. SELinux Denials

**Symptom**: Permission errors in logs on Fedora Atomic

**Solution**:
```bash
# Check for denials
sudo ausearch -m avc -ts recent

# The :z flag in the Volume directive should handle this
# If issues persist, check SELinux context:
ls -lZ /home/rhlabs/podman/volumes/openvpn-config/
```

### Performance Tuning

#### Optimize for Old Hardware

```bash
# In the Admin UI, reduce encryption overhead:
# Configuration > Advanced VPN:
# - Use AES-128-CBC instead of AES-256-CBC
# - Reduce compression if CPU is weak

# Limit concurrent connections in User Management
```

#### Network Optimization

```bash
# For better throughput on stable connections
# Configuration > Network Settings:
# - Increase MTU if your network supports it
# - Use TCP if UDP packets are being dropped
```

------------------------------------------------------------

## Part 5: Security Best Practices

### 1. Regular Updates

```bash
# Set up automatic image updates (optional)
# Add to Quadlet file:
# AutoUpdate=registry

# Enable podman auto-update timer
sudo systemctl enable --now podman-auto-update.timer
```

### 2. Strong Authentication

- Use strong passwords (minimum 12 characters)
- Enable two-factor authentication
- Regularly rotate passwords
- Disable auto-login profiles for sensitive deployments

### 3. Access Control

- Limit VPN access to specific IP ranges when possible
- Use user groups to manage permissions
- Regularly audit user accounts and remove unused accounts
- Monitor connection logs for suspicious activity

### 4. Network Segmentation

- Consider routing VPN users to a separate subnet
- Use firewall rules to limit access to sensitive systems
- Implement the principle of least privilege

### 5. Certificate Security

- Use proper SSL certificates for the web interface
- Regularly renew certificates before expiration
- Keep certificate private keys secure

------------------------------------------------------------

## Conclusion

You've now successfully transformed an old computer into a powerful VPN server running OpenVPN Access Server on Fedora Budgie Atomic. This setup provides:

✅ **Secure remote access** to your home network  
✅ **Enterprise-grade VPN** with web management  
✅ **Modern containerized deployment** using Podman Quadlet  
✅ **Support for all major client platforms**  
✅ **Easy maintenance and updates**

### Next Steps

- Configure split-tunneling for optimal performance
- Set up Let's Encrypt certificates for the web interface
- Explore advanced routing scenarios
- Implement monitoring and alerting
- Consider setting up high availability with multiple servers

### Additional Resources

- **OpenVPN Access Server Documentation**: https://openvpn.net/as-docs/v3/getting-started.html
- **Podman Quadlet Documentation**: https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html
- **Fedora Atomic Documentation**: https://docs.fedoraproject.org/en-US/fedora-silverblue/
- **OpenVPN Community Forums**: https://forums.openvpn.net/

### Questions or Issues?

If you encounter problems or have suggestions for improving this guide, feel free to reach out or leave a comment below!

---

**Happy VPN-ing! 🔒**
