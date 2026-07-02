# SOC Lab — Linux to Windows Log Forwarding

A hands-on lab demonstrating how to forward logs from a Linux machine to a Windows log collector using the syslog protocol. This mirrors how real SOC environments aggregate logs from multiple sources before ingestion into a SIEM.

---

## Lab Architecture

```
Ubuntu Server VM (Log Source)
        |
        |  rsyslog — UDP port 514
        |  (syslog protocol)
        ↓
Windows Host (Log Collector)
Kiwi Syslog Server NG
```

---

## Tools Used

| Tool | Role |
|------|------|
| VMware Workstation | Hypervisor to run the Linux VM |
| Ubuntu Server 24.04 | Log source (Linux VM) |
| rsyslog | Log forwarding daemon on Linux |
| Kiwi Syslog Server NG | Log collector on Windows |
| Windows Firewall | Port 514 opened for inbound syslog traffic |

---

## Setup

### Linux VM (Ubuntu Server on VMware)

- Network adapter set to NAT (VMnet8)
- IP assigned: `192.168.125.129`
- rsyslog installed by default on Ubuntu Server

### Windows Host (Log Collector)

- IP on VMware NAT network: `192.168.125.1`
- Kiwi Syslog Server NG installed and running as a service
- Listening on UDP port 514

---

## Configuration

### rsyslog forward rule (Linux)

File: `/etc/rsyslog.d/50-forward.conf`

```
*.* @192.168.125.1:514
```

- `*.*` — forward all facilities and all severity levels
- `@` — use UDP protocol (single @ = UDP, @@ = TCP)
- `192.168.125.1` — IP of the Windows log collector
- `514` — standard syslog port

Restart rsyslog after adding the config:

```bash
sudo systemctl restart rsyslog
sudo systemctl status rsyslog
```

### Windows Firewall rules

Run in PowerShell as Administrator:

```powershell
New-NetFirewallRule -DisplayName "Kiwi Syslog UDP 514" -Direction Inbound -Protocol UDP -LocalPort 514 -Action Allow
New-NetFirewallRule -DisplayName "Kiwi Syslog TCP 514" -Direction Inbound -Protocol TCP -LocalPort 514 -Action Allow
New-NetFirewallRule -DisplayName "Allow VMware NAT Subnet" -Direction Inbound -RemoteAddress 192.168.125.0/24 -Action Allow
```

---

## Testing

### 1. Custom log message

```bash
logger "Test syslog from Linux SOC lab to Windows"
```

### 2. Simulated brute force — failed SSH login attempts

```bash
ssh baduser@localhost
# Enter wrong password 3-4 times
```

This generates `auth` facility logs — failed password attempts for an invalid user. In a real SOC this pattern would trigger a brute force detection alert mapped to **MITRE ATT&CK T1110 (Brute Force)**.

### 3. Privilege escalation activity

```bash
sudo cat /etc/shadow
logger -p auth.warning "Suspicious login attempt detected"
```

---

## Result

All log events from Linux appeared in real time on Kiwi Syslog Server on the Windows side with full metadata:

- **Source IP:** `192.168.125.129` (Linux VM)
- **Protocol:** UDP
- **Facility:** auth / syslog
- **Severity:** Info / Warning
- **Message:** actual log content from Linux

---

## SOC Relevance

This lab demonstrates the foundational log collection pipeline used in real MSSP and SOC environments:

- Endpoints and servers forward logs via syslog (UDP/TCP 514) to a central collector
- The collector aggregates and normalises logs before feeding them into a SIEM (Splunk, QRadar, etc.)
- Failed SSH login events like those generated here are a primary indicator for brute force detection rules in any SIEM

---

## MITRE ATT&CK Mapping

| Technique | ID | Description |
|-----------|-----|-------------|
| Brute Force | T1110 | Simulated via repeated failed SSH login attempts |

---

## Author

Hardee — SOC Intern, Talakunchi Networks  
GitHub: [Hardeeh03](https://github.com/Hardeeh03)
