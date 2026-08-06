---
title: How to Configure Windows RADIUS and UniFi Controller
url: https://devopstales.github.io/windows/windows-server-radius/
date: 2026-03-12
keywords: Windows Server 2022, Windows Server 2025, RADIUS, NPS, UniFi Controller, 802.1X, WPA-Enterprise, wireless authentication
---


Implementing 802.1X wireless authentication with Windows NPS (Network Policy Server) and UniFi access points provides enterprise-grade security for your wireless network. This updated guide for 2026 covers Windows Server 2022/2025 and the latest UniFi Controller.

<!--more-->

## Architecture Overview

```bash
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │◄──►│   UniFi     │◄──►│  Windows    │◄──►│   Active    │
│ (Supplicant)│    │     AP      │    │    NPS      │    │  Directory  │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
     802.1X              RADIUS            LDAP/AD
    EAPOL               UDP 1812
```

## Prerequisites

- **Windows Server 2022/2025** (Domain Controller or Member Server)
- **Active Directory Domain** with user accounts
- **UniFi Controller** (self-hosted or cloud key)
- **UniFi Access Points** with latest firmware
- **Network connectivity** between APs and NPS server

## Install the NPS Server Role

### Using Server Manager

1. Open **Server Manager** → **Manage** → **Add Roles and Features**
2. Click **Next** through the Before You Begin screen
3. Select **Role-based or feature-based installation**
4. Select your server from the pool
5. Check **Network Policy and Access Services**

![Add Roles Wizard](/img/include/radius001.webp)

6. Click **Add Features** when prompted
7. Click **Next** through the Features screen

![NPS Role](/img/include/radius002.webp)

8. On the **Network Policy and Access Services** screen, click **Next**
9. Ensure **Network Policy Server** is checked under Role Services

![NPS Role Services](/img/include/radius003.webp)

10. Click **Next** and then **Install**

![Install NPS](/img/include/radius004.webp)

### Using PowerShell

```powershell
# Install NPS role
Install-WindowsFeature NPAS -IncludeManagementTools

# Verify installation
Get-WindowsFeature NPAS

# Register NPS in Active Directory (if not done automatically)
Register-Server -Server "localhost"
```

## Configure Firewall Rules

### Using Windows Defender Firewall

Open **Windows Defender Firewall with Advanced Security** and create inbound rules:

1. **New Rule** → **Port** → **Next**
2. Select **TCP** and enter port **1812**
3. Select **Allow the connection**
4. Apply to appropriate profiles (Domain recommended)
5. Name: "RADIUS Authentication (TCP)"

![Firewall Rule 1](/img/include/radius019.webp)
![Firewall Rule 2](/img/include/radius020.webp)

Repeat for:
- **UDP 1812** (RADIUS Authentication)
- **UDP 1813** (RADIUS Accounting)
- **TCP 1813** (RADIUS Accounting - optional)

![Firewall Rule 3](/img/include/radius021.webp)
![Firewall Rule 4](/img/include/radius022.webp)

### Using PowerShell

```powershell
# Create firewall rules
New-NetFirewallRule -DisplayName "RADIUS Authentication UDP" `
  -Direction Inbound -Protocol UDP -LocalPort 1812 -Action Allow

New-NetFirewallRule -DisplayName "RADIUS Accounting UDP" `
  -Direction Inbound -Protocol UDP -LocalPort 1813 -Action Allow

New-NetFirewallRule -DisplayName "RADIUS Authentication TCP" `
  -Direction Inbound -Protocol TCP -LocalPort 1812 -Action Allow

New-NetFirewallRule -DisplayName "RADIUS Accounting TCP" `
  -Direction Inbound -Protocol TCP -LocalPort 1813 -Action Allow

# Verify rules
Get-NetFirewallRule | Where-Object {$_.DisplayName -like "*RADIUS*"}
```

## Configure NPS Server

### Launch NPS Console

Open **Network Policy Server** from Server Manager or run:

```powershell
nps.msc
```

### Register NPS in Active Directory

If not already done:

1. Right-click **NPS (Local)** in the left pane
2. Select **Register server in Active Directory**
3. Click **OK** to confirm

![Register NPS](/img/include/radius005.webp)

### Configure 802.1X Wireless Network

1. In the NPS console, click **NPS (Local)**
2. Select **RADIUS Clients and Servers** → **RADIUS Clients**
3. Right-click **RADIUS Clients** → **New**

![RADIUS Clients](/img/include/radius012.webp)

### Add UniFi Access Points as RADIUS Clients

For each UniFi AP (or use a network range):

| Field | Value |
|-------|-------|
| **Friendly name** | AP-Floor1-01 (descriptive name) |
| **Manufacturer** | UniFi (or Any) |
| **Address (IP or DNS)** | 192.168.1.10 (AP IP address) |
| **Shared secret** | [Generate a strong secret] |
| **Confirm shared secret** | [Same secret] |

![Add RADIUS Client](/img/include/radius013.webp)
![RADIUS Client Config](/img/include/radius014.webp)

**Best Practices:**
- Use a unique shared secret per AP or AP group
- Minimum 22 characters with mixed case, numbers, and symbols
- Store secrets securely (password manager)

### Configure Network Policies

1. Right-click **Network Policies** → **New**
2. Select **RADIUS server for 802.1X Wireless or Wired Connections**

![New Policy Wizard](/img/include/radius015.webp)

#### Policy Configuration

**Specify 802.1X Switch or Wireless Access Point:**
- Select your APs from the list or add new ones

**Specify RADIUS Client:**
- Select the RADIUS clients you configured

**Configure Authentication Methods:**

![Authentication Methods](/img/include/radius016.webp)

Recommended settings for 2026:

| Method | Setting |
|--------|---------|
| **Microsoft: Protected EAP (PEAP)** | Enabled |
| **Microsoft: Smart Card or other certificate** | Enabled |
| **EAP Types** | EAP-TLS (recommended) or PEAP-MSCHAPv2 |

**For PEAP-MSCHAPv2:**
- Uncheck "Enable Fast Reconnect"
- Click "Configure" and ensure server certificate is selected
- Uncheck "Enable Identity Privacy" unless required

**For EAP-TLS (Recommended):**
- Select your CA certificate
- Enable "Use Windows Hello for Business" if applicable

#### Configure Constraints

![Policy Constraints](/img/include/radius017.webp)

| Constraint | Recommended Setting |
|------------|---------------------|
| **Authentication Methods** | PEAP or EAP-TLS |
| **Idle Timeout** | 30 minutes |
| **Session Timeout** | 8 hours |
| **Called Station ID** | Restrict to specific SSIDs if needed |
| **NAS Port Type** | Wireless - IEEE 802.11 |

#### Configure Settings

![Policy Settings](/img/include/radius018.webp)

| Setting | Configuration |
|---------|---------------|
| **Standard** | Log requests, discard if unable |
| **RADIUS Attributes** | Add vendor-specific attributes as needed |
| **RADIUS Authentication** | Ignore NAS-Port-Type if needed |

## Configure UniFi Controller

### Access UniFi Network Application

Navigate to your UniFi Controller (`https://controller.unifi.local/`) and log in.

### Create Wireless Network

1. Go to **Settings** → **WiFi**
2. Click **Create New WiFi Network**

![UniFi WiFi Settings](/img/include/radius026.webp)

### Configure SSID

| Setting | Value |
|---------|-------|
| **Name (SSID)** | Enterprise-WiFi |
| **Security** | WPA Enterprise |
| **Authentication** | RADIUS |
| **Enable Security Settings** | Yes |

### Create RADIUS Profile

1. Click **Create New RADIUS Profile**
2. Configure the following:

![RADIUS Profile](/img/include/radius027.webp)

| Field | Value |
|-------|-------|
| **Profile Name** | Corporate-NPS |
| **Authentication Server** | 192.168.1.50 (NPS server IP) |
| **Authentication Port** | 1812 |
| **Authentication Secret** | [Your shared secret] |
| **Accounting Server** | 192.168.1.50 |
| **Accounting Port** | 1813 |
| **Accounting Secret** | [Your shared secret] |

![RADIUS Settings](/img/include/radius028.webp)

### Advanced Security Settings

For enhanced security in 2026:

| Setting | Recommended |
|---------|-------------|
| **PMF (Protected Management Frames)** | Required |
| **WPA3 Transition Mode** | Enabled (if clients support) |
| **Minimum Data Rate Control** | Disable legacy rates (1, 2, 5.5, 11 Mbps) |
| **Fast Roaming** | Enabled (802.11r) |
| **UAPSD** | Enabled for voice optimization |

### Apply Configuration

1. Review settings
2. Click **Apply Changes**
3. Wait for APs to provision (may take 2-5 minutes)

## Test Wireless Authentication

### Windows Client

1. Connect to the Enterprise-WiFi network
2. Enter domain credentials when prompted
3. Verify connection succeeds

```powershell
# View wireless profiles
netsh wlan show profiles

# View detailed profile info
netsh wlan show profile name="Enterprise-WiFi" key=clear

# Test authentication
netsh wlan connect name="Enterprise-WiFi"
```

### View NPS Logs

```powershell
# View NPS logs
Get-EventLog -LogName "Security" -Source "IAS" -Newest 50

# Or use Event Viewer:
# Event Viewer → Windows Logs → Security → Filter by Source: IAS
```

### UniFi Controller Logs

In UniFi Controller:
1. Go to **Insights** or **Logs**
2. Filter for authentication events
3. Verify RADIUS authentication success

## Troubleshooting

### Common Issues

#### "The network password needs to be entered again"

**Cause:** Incorrect shared secret or RADIUS server unreachable

**Solution:**
- Verify shared secret matches on both NPS and UniFi
- Check firewall rules allow UDP 1812/1813
- Test connectivity: `Test-NetConnection <NPS-IP> -Port 1812`

#### "Can't connect to this network"

**Cause:** Certificate issues or EAP type mismatch

**Solution:**
- Verify NPS server has valid certificate
- Check EAP types match between NPS and client
- For PEAP, ensure server certificate is trusted

#### Authentication succeeds but no network access

**Cause:** VLAN configuration or authorization issues

**Solution:**
- Check NPS network policy allows the user group
- Verify VLAN assignment in UniFi
- Check switch port configuration

### NPS Diagnostic Commands

```powershell
# Check NPS service status
Get-Service IAS

# Restart NPS service
Restart-Service IAS

# View NPS configuration
Get-NpsConfiguration

# Test RADIUS client connectivity
Test-NetConnection -ComputerName <NPS-IP> -Port 1812
```

### UniFi Diagnostic Commands

```bash
# SSH to UniFi AP
ssh admin@<ap-ip>

# View RADIUS statistics
cat /proc/sys/net/ipv4/conf/all/accept_redirects

# Check AP logs
tail -f /var/log/messages | grep -i radius
```

## Security Best Practices for 2026

### Certificate-Based Authentication (EAP-TLS)

Migrate from password-based (PEAP-MSCHAPv2) to certificate-based authentication:

1. Deploy certificates to all devices via Intune or Group Policy
2. Configure NPS to require EAP-TLS
3. Disable password-based methods

### NPS Hardening

```powershell
# Enable NPS logging
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\RemoteAccess\Parameters\LogFileDirectories" `
  -Name "LogFiles" -Value "C:\Windows\System32\LogFiles\NPS"

# Configure account lockout policy
net accounts /lockoutthreshold:5 /lockoutduration:30
```

### Network Segmentation

- Place wireless clients on separate VLAN
- Implement network access control (NAC)
- Use firewall rules to restrict wireless client access

### Monitoring and Alerting

```powershell
# Create scheduled task to monitor failed authentications
$action = New-ScheduledTaskAction -Execute "PowerShell.exe" `
  -Argument "-Command Get-EventLog -LogName Security -Source IAS -EntryType FailureAudit -Newest 10"
$trigger = New-ScheduledTaskTrigger -Daily -At 8am
Register-ScheduledTask -TaskName "NPS-Failure-Monitor" -Action $action -Trigger $trigger
```

## Windows Server 2025 Enhancements

Windows Server 2025 introduces:

- **Enhanced NPS logging** with more detailed authentication data
- **Improved certificate validation** for EAP-TLS
- **Better integration with Azure AD** for hybrid scenarios
- **TLS 1.3 support** for RADIUS over TLS (RadSec)

