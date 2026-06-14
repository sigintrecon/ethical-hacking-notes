# 07 — OS Fingerprinting

> **Module:** Network Enumeration
> **Difficulty:** Beginner
> **RTO Relevance:** Medium — Knowing the OS guides exploit selection and attack approach

---

## What is OS Fingerprinting?

OS Fingerprinting is the process of **identifying the operating system** running on a remote target — without logging in. By analyzing how a system responds to specific network packets, attackers (and defenders) can determine the OS type and version.

```
Attacker sends crafted packets
    ↓
Target responds (different OSes respond differently)
    ↓
Response patterns analyzed
    ↓
OS identified: "Windows Server 2019 Build 17763"
```

---

## Two Types of Fingerprinting

### Active Fingerprinting
You **actively send packets** to the target to analyze responses.

```bash
# Nmap OS detection (sends crafted packets)
nmap -O 192.168.1.10

# Aggressive scan (OS + version + scripts + traceroute)
nmap -A 192.168.1.10
```

**Pros:** More accurate results
**Cons:** Generates traffic — can be detected by IDS/IPS

### Passive Fingerprinting
You **capture and analyze existing traffic** without sending any packets.

```bash
# p0f — passive OS fingerprinting tool
p0f -i eth0

# Analyzes packets passing through your interface
# No packets sent — completely silent
```

**Pros:** Completely silent — undetectable
**Cons:** Must have network access to capture traffic

---

## What OS Fingerprinting Analyzes

Different operating systems have unique characteristics in their network responses:

| Characteristic | What It Reveals |
|---------------|----------------|
| TTL values | Windows (128), Linux (64), Cisco (255) |
| TCP window size | OS-specific default values |
| TCP flags | Different OSes set flags differently |
| ICMP responses | Error message formats vary by OS |
| Port behavior | Which ports open/closed/filtered |
| Banner grabbing | Service banners often reveal OS |

---

## Red Team Perspective

### Basic OS Detection with Nmap

```bash
# OS detection requires root/sudo
sudo nmap -O 192.168.1.10

# Output example:
# Running: Microsoft Windows 2019
# OS CPE: cpe:/o:microsoft:windows_server_2019
# OS details: Windows Server 2019 Build 17763
```

### Version Detection + OS Combined

```bash
# -sV = service version detection
# -O = OS detection
# Together they paint a complete target picture
sudo nmap -sV -O 192.168.1.10

# Output tells you:
# Port 445 → SMB → Windows Server 2019
# Port 88  → Kerberos → Domain Controller confirmed
# Port 389 → LDAP → Active Directory running
# Port 3389 → RDP → Remote Desktop enabled
```

### Banner Grabbing — Simple Version Detection

```bash
# Netcat banner grab
nc -v 192.168.1.10 22
# SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.5
# ← Reveals: Linux Ubuntu with specific OpenSSH version

# Telnet banner grab
telnet 192.168.1.10 80
# Server: Apache/2.4.41 (Ubuntu)
# ← Reveals: Web server type and OS
```

### TTL-Based Quick OS Identification

Without any tools, you can guess the OS just from a `ping`:

```bash
ping 192.168.1.10

# TTL=128 → Almost certainly Windows
# TTL=64  → Almost certainly Linux
# TTL=255 → Cisco router or network device
```

### RTO Thought Process After OS Fingerprinting

```
Target identified: Windows Server 2019
    ↓
Is it a Domain Controller?
(Check ports 88, 389, 3268)
    ↓ Yes
Kerberos attacks possible
LDAP enumeration possible
    ↓
What patches are missing?
(Version fingerprinting → CVE lookup)
    ↓
Select appropriate attack path
```

### Evading OS Fingerprinting (Defender Tricks)

Defenders can configure systems to **lie** about their OS:

```
Real OS: Windows Server 2019
Configured to respond with: Linux TTL values
    ↓
Attacker's Nmap thinks it's Linux
    ↓
Wrong exploits tried → Attack fails
```

---

## Defense Perspective

| Defense | How It Helps |
|---------|-------------|
| Customize TTL values | Mislead passive fingerprinting |
| Firewall ICMP | Blocks ping-based fingerprinting |
| Disable unnecessary banners | Reduces information disclosure |
| IDS/IPS rules | Detect active fingerprinting scans |
| Honeypots | Fake OS responses to mislead attackers |

---

## Key Takeaways

- OS Fingerprinting = Identifying remote OS without logging in
- **Active:** Send packets, analyze responses (Nmap `-O`) — detectable
- **Passive:** Capture existing traffic (p0f) — completely silent
- TTL values: Windows=128, Linux=64, Cisco=255
- Banner grabbing = quick version/OS identification
- OS knowledge guides **exploit selection** and **attack path planning**
- Tool: `nmap -O` or `nmap -A` for comprehensive fingerprinting

---

*← [NFS/RPC](06-nfs-rpc.md) | Back to [README](../../README.md) | Next Module: [AD Fundamentals →](../02-active-directory/01-ad-fundamentals.md)*
