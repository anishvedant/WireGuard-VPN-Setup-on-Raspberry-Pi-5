# 🔒 WireGuard VPN Setup on Raspberry Pi 5 with Pi-hole Integration

A complete guide to setting up a secure VPN server on Raspberry Pi 5, allowing remote access to your home network with ad-blocking via Pi-hole.

## 📋 Table of Contents
- [Project Overview](#-project-overview)
- [System Architecture](#-system-architecture)
- [Prerequisites](#-prerequisites)
- [Installation Process](#-installation-process)
- [Configuration](#-configuration)
- [Client Setup](#-client-setup)
- [Testing & Verification](#-testing--verification)
- [Troubleshooting](#-troubleshooting)
- [Maintenance](#-maintenance)

## 🎯 Project Overview

This project implements a WireGuard VPN server on Raspberry Pi 5 using PiVPN, integrated with Pi-hole for DNS filtering. The setup provides secure remote access to the home network from anywhere in the world.

### Key Features
- **WireGuard Protocol**: Modern, fast, and secure VPN
- **Pi-hole Integration**: Network-wide ad blocking even when remote
- **Easy Management**: Simple client creation and revocation via PiVPN
- **Split/Full Tunnel Support**: Flexible routing configurations

## 🏗 System Architecture

```
Internet ←→ Router (Public IP) ←→ Raspberry Pi (10.0.0.53) ←→ Home LAN (10.0.0.0/24)
                ↓                            ↓
          Port Forward                 WireGuard (VPN Subnet)
           UDP 51820                    + Pi-hole DNS
```

### Network Details
- **Home Network**: 10.0.0.0/24 (typical home network range)
- **VPN Subnet**: Automatically assigned by PiVPN
- **Pi Static IP**: 10.0.0.53
- **WireGuard Port**: 51820/UDP
- **DNS Server**: VPN gateway IP (assigned by PiVPN)

## ✅ Prerequisites

### Hardware
- Raspberry Pi 5 (4GB/8GB RAM)
- MicroSD card (32GB+)
- Ethernet connection
- Reliable power supply

### Software
- Raspberry Pi OS (64-bit)
- SSH access enabled
- Pi-hole installed and configured
- Admin access to router

## 📦 Installation Process

### Step 1: System Preparation

First, ensure the system is up to date:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl wget git vim htop net-tools tcpdump
```

### Step 2: Set Static IP for Raspberry Pi

#### Option A: DHCP Reservation (Recommended)
1. Access router admin panel: `http://10.0.0.1`
2. Navigate to Connected Devices
3. Find your Raspberry Pi
4. Reserve IP address: `10.0.0.53`

#### Option B: Manual Configuration
```bash
sudo nano /etc/dhcpcd.conf

# Add these lines:
interface eth0
static ip_address=10.0.0.53/24
static routers=10.0.0.1
static domain_name_servers=10.0.0.53
```

### Step 3: PiVPN Installation

```bash
curl -L https://install.pivpn.io | bash
```

#### Installation Wizard Options:

1. **Choose User**: Select your username
2. **VPN Type**: Select **WireGuard**
3. **Port**: Accept default **51820/UDP**
4. **DNS Provider**: Choose **Custom** → Enter VPN gateway IP (shown during setup)
5. **Public IP/DNS**: Select **Public IP Address** (auto-detected)
6. **Unattended Updates**: Enable for security

The installer will:
- Check for existing packages
- Install WireGuard and dependencies
- Configure IP forwarding
- Set up firewall rules
- Create initial configuration

### Step 4: Verify Installation

```bash
# Check WireGuard status
sudo wg show

# Verify service is running
sudo systemctl status wg-quick@wg0

# Run debug check
pivpn -d
```

Expected output:
```
interface: wg0
  public key: [SERVER_PUBLIC_KEY]
  private key: (hidden)
  listening port: 51820
```

## ⚙ Configuration

### Router Port Forwarding

#### Using Router Web Interface:
1. Navigate to `http://10.0.0.1` (or your router's IP)
2. Go to Advanced → Port Forwarding
3. Add new rule:
   - **Service Name**: WireGuard
   - **Server IPv4**: 10.0.0.53
   - **External Port**: 51820
   - **Internal Port**: 51820
   - **Protocol**: UDP

#### Using Router Mobile App:
1. Open your router's app
2. Advanced Settings → Port Forwarding
3. Add Port Forward:
   - **Device**: Raspberry Pi (10.0.0.53)
   - **Port**: 51820
   - **Protocol**: UDP

### Creating Client Profiles

```bash
# Create new client
pivpn -a
```

When prompted:
- **Client IP**: Press Enter for auto-assign
- **Client Name**: Enter descriptive name (e.g., "windows-laptop")

Output:
```
::: Client Keys generated
::: Client config generated
::: Updated server config
::: Updated hosts file for Pi-hole
::: WireGuard reloaded
======================================================================
::: Done! windows-laptop.conf successfully created!
::: windows-laptop.conf was copied to /home/[username]/configs for easy transfer.
======================================================================
```

### Client Configuration Structure

The generated configuration file contains:

```ini
[Interface]
PrivateKey = [GENERATED_PRIVATE_KEY]
Address = [VPN_CLIENT_IP]/24
DNS = [VPN_DNS_SERVER]

[Peer]
PublicKey = [SERVER_PUBLIC_KEY]
PresharedKey = [GENERATED_PRESHARED_KEY]
Endpoint = [YOUR_PUBLIC_IP]:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

### Modifying for Split-Tunnel

For accessing only home network (recommended):

```ini
[Interface]
PrivateKey = [KEEP_EXISTING]
Address = [VPN_CLIENT_IP]/24
DNS = 10.0.0.53  # Use Pi-hole directly

[Peer]
PublicKey = [KEEP_EXISTING]
PresharedKey = [KEEP_EXISTING]
Endpoint = [YOUR_PUBLIC_IP]:51820
AllowedIPs = 10.0.0.0/24, [VPN_SUBNET]/24  # Only home and VPN networks
PersistentKeepalive = 25
```

### Finding Your Public IP

To find your public IP for the configuration:
```bash
# From the Pi or any device on your network
curl -s https://ipinfo.io/ip
```

## 💻 Client Setup

### Windows WireGuard Client

#### Step 1: Transfer Configuration

Using PowerShell:
```powershell
scp [username]@10.0.0.53:/home/[username]/configs/[clientname].conf $env:USERPROFILE\Downloads\
```

#### Step 2: Install WireGuard

Download from: [https://www.wireguard.com/install/](https://www.wireguard.com/install/)

Or use PowerShell:
```powershell
winget install WireGuard.WireGuard
```

#### Step 3: Import Configuration

1. Open WireGuard application
2. Click "Add Tunnel" → "Import from file"
3. Select downloaded `.conf` file
4. Click "Activate" to connect

## 🧪 Testing & Verification

### Local Network Test

When on home network:
```cmd
# Test connectivity to Pi
ping 10.0.0.53

# Expected output:
Reply from 10.0.0.53: bytes=32 time<10ms TTL=64
Ping statistics: 0% loss
```

### Remote Connection Test

1. **Switch to Different Network**
   - Use mobile hotspot
   - Or connect to different WiFi
   - Ensure you're not on home network

2. **Activate VPN**
   - Open WireGuard
   - Click "Activate" on your tunnel
   - Status should show "Active"

3. **Verify Connection**
```cmd
# Check VPN interface
ipconfig /all

# Should show WireGuard adapter with VPN IP

# Test connectivity
ping 10.0.0.53
```

### Connection Statistics

After successful connection:
- **Latest Handshake**: Updates every ~25-60 seconds
- **Transfer**: Shows data sent/received
- **Status**: Active

### Server-Side Verification

```bash
# View active connections
pivpn -c

# Output example:
Name         Remote IP    Virtual IP    Bytes Received    Bytes Sent    Last Seen
client1      [HIDDEN]     [VPN_IP]      628KiB           876KiB        [TIMESTAMP]

# Monitor in real-time
sudo wg show
```

## 🔧 Troubleshooting

### Issue 1: Missing MASQUERADE Rule

**Symptom**: `pivpn -d` shows error about missing iptables rule

**Solution**:
```bash
# Let PiVPN fix it automatically
pivpn -d
# When prompted about MASQUERADE rule, type: Y

# Or manually add:
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo netfilter-persistent save
```

### Issue 2: Internet Drops When Connected

**Problem**: Full tunnel with IPv6 issues

**Solution**: Remove IPv6 from AllowedIPs:
```ini
# Change from:
AllowedIPs = 0.0.0.0/0, ::/0

# To (for split-tunnel):
AllowedIPs = 10.0.0.0/24, [VPN_SUBNET]/24

# Or (for full-tunnel IPv4 only):
AllowedIPs = 0.0.0.0/0
```

### Issue 3: DNS Not Working

**Solution**: Configure Pi-hole properly:
1. Access Pi-hole admin: `http://10.0.0.53/admin`
2. Settings → DNS
3. Interface settings: "Listen on all interfaces, permit all origins"
4. Save settings

### Issue 4: Can't Connect from Outside

**Checklist**:
- ✅ Port 51820/UDP forwarded to 10.0.0.53
- ✅ WireGuard service running: `sudo systemctl status wg-quick@wg0`
- ✅ Public IP correct in client config
- ✅ Testing from different network (not home WiFi)
- ✅ No double NAT (check if router WAN IP is private)

## 🛠 Maintenance

### Regular Tasks

```bash
# System updates
sudo apt update && sudo apt upgrade -y
pivpn -up

# Client management
pivpn -l          # List all clients
pivpn -a          # Add new client
pivpn -r NAME     # Remove client
pivpn -qr NAME    # Generate QR code for mobile

# Service management
sudo systemctl restart wg-quick@wg0    # Restart VPN
sudo wg-quick down wg0                 # Stop VPN
sudo wg-quick up wg0                   # Start VPN
```

### Monitoring Commands

```bash
# View connection status
sudo wg show

# Watch active connections
watch -n 1 sudo wg show

# Monitor traffic
sudo tcpdump -nni any udp port 51820

# Check logs
sudo journalctl -u wg-quick@wg0 -f
```

### Backup Configuration

```bash
# Backup WireGuard configs
sudo cp -r /etc/wireguard ~/wireguard-backup-$(date +%Y%m%d)

# Backup client configs
cp -r ~/configs ~/configs-backup-$(date +%Y%m%d)

# Backup PiVPN
pivpn -bk
```

## 📊 Performance Metrics

Based on real-world testing:

| Metric | Local Network | Remote Connection |
|--------|--------------|-------------------|
| Latency | < 10ms | 50-150ms |
| Throughput | Full speed | Varies by connection |
| Handshake Time | < 1 second | < 2 seconds |
| Stability | 100% | 99%+ |
| Packet Loss | 0% | < 1% |

## 🔐 Security Best Practices

1. **Regular Updates**
   ```bash
   # Enable automatic security updates
   sudo apt install unattended-upgrades
   sudo dpkg-reconfigure unattended-upgrades
   ```

2. **Strong Keys**: PiVPN generates cryptographically secure keys automatically

3. **Firewall Rules**: Only expose necessary ports
   ```bash
   sudo ufw allow 51820/udp
   sudo ufw allow from 10.0.0.0/24
   sudo ufw enable
   ```

4. **Client Isolation**: Each client gets unique keys

5. **Revoke Compromised Clients**
   ```bash
   pivpn -r compromised-client-name
   ```

## 📝 Configuration Files Reference

### Server: `/etc/wireguard/wg0.conf`
```ini
[Interface]
PrivateKey = [SERVER_PRIVATE_KEY]
Address = [VPN_GATEWAY_IP]/24
ListenPort = 51820
MTU = 1420

### begin client ###
[Peer]
PublicKey = [CLIENT_PUBLIC_KEY]
PresharedKey = [PRESHARED_KEY]
AllowedIPs = [CLIENT_VPN_IP]/32
### end client ###
```

### Client: `~/configs/client.conf`
```ini
[Interface]
PrivateKey = [CLIENT_PRIVATE_KEY]
Address = [CLIENT_VPN_IP]/24
DNS = [VPN_DNS_SERVER]

[Peer]
PublicKey = [SERVER_PUBLIC_KEY]
PresharedKey = [PRESHARED_KEY]
Endpoint = [PUBLIC_IP]:51820
AllowedIPs = 0.0.0.0/0, ::/0
```

## 🎉 Summary

Successfully deployed WireGuard VPN on Raspberry Pi 5 with:
- ✅ Secure remote access from any network
- ✅ Pi-hole DNS filtering integration
- ✅ Easy client management via PiVPN
- ✅ Both split and full tunnel support
- ✅ Stable performance with minimal latency

The setup provides enterprise-grade VPN functionality with complete control over your data and network privacy.

## 📚 Additional Resources

- [PiVPN Documentation](https://docs.pivpn.io/)
- [WireGuard Quick Start](https://www.wireguard.com/quickstart/)
- [Pi-hole Documentation](https://docs.pi-hole.net/)
- [PiVPN FAQ](https://docs.pivpn.io/faq/)
- [WireGuard White Paper](https://www.wireguard.com/papers/wireguard.pdf)

---

### Optional: Dynamic DNS Setup

If your public IP changes frequently, consider setting up Dynamic DNS:

1. **Create DuckDNS Account**
   - Visit [duckdns.org](https://www.duckdns.org)
   - Create subdomain
   - Note your token

2. **Configure Auto-Update**
   ```bash
   mkdir ~/duckdns
   cd ~/duckdns
   
   cat > duck.sh << 'EOF'
   #!/bin/bash
   echo url="https://www.duckdns.org/update?domains=YOURSUBDOMAIN&token=YOURTOKEN&ip=" | curl -k -o ~/duckdns/duck.log -K -
   EOF
   
   chmod +x duck.sh
   
   # Add to crontab
   (crontab -l 2>/dev/null; echo "*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1") | crontab -
   ```

3. **Update Client Configs**
   - Replace public IP with `yoursubdomain.duckdns.org`

---

### Quick Command Reference

```bash
# Find your public IP
curl -s https://ipinfo.io/ip

# Check VPN status
sudo wg show

# List clients
pivpn -c

# Add new client
pivpn -a

# Debug issues
pivpn -d

# View logs
sudo journalctl -u wg-quick@wg0 -f
```

---

*Documentation Version: 1.0*  
*Last Updated: September 2025*  
*Tested on: Raspberry Pi 5, Raspberry Pi OS (64-bit), WireGuard via PiVPN*
