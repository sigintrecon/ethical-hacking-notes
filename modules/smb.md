# 03 — SMB (Server Message Block)

> **Module:** Network Enumeration
> **Difficulty:** Beginner-Intermediate
> **RTO Relevance:** Critical — SMB is the backbone of Windows lateral movement

---

## What is SMB?

SMB is a **file sharing and communication protocol** used primarily in Windows environments. It allows computers to share files, printers, and other resources across a network.

Every time you access `\\server\share` on a Windows machine — that is SMB working.

---

## How SMB Works

```
Client (ali.hassan's PC)          Server (FILE-SERVER)
        │                                │
        │── "I want to access            │
        │    \\FILE-SERVER\Reports" ───→ │
        │                                │
        │ ←── "Authenticate first" ───── │
        │                                │
        │── Sends credentials ─────────→ │
        │                                │
        │ ←── "Access granted" ────────  │
        │                                │
        │ ←── File data transferred ──── │
```

---

## Important Ports

| Port | Protocol | Usage |
|------|----------|-------|
| TCP 139 | NetBIOS over SMB | Legacy SMB (older systems) |
| TCP 445 | SMB Direct | Modern SMB (Windows 2000+) |

---

## SMB Versions

| Version | OS | Security |
|---------|----|---------|
| SMBv1 | Windows XP/2003 | ❌ Extremely dangerous — EternalBlue |
| SMBv2 | Windows Vista/2008 | 🟡 Better but still has issues |
| SMBv3 | Windows 8/2012+ | ✅ Encryption supported |

> **RTO Note:** Finding SMBv1 enabled = potential EternalBlue (MS17-010) exploit opportunity.

---

## Red Team Perspective

### Phase 1 — SMB Enumeration

```bash
# Nmap SMB scan
nmap -p 445 --script smb-enum-shares,smb-enum-users 192.168.1.0/24

# enum4linux — comprehensive SMB enumeration
enum4linux -a 192.168.1.50

# What enum4linux reveals:
# - OS version and hostname
# - Domain/Workgroup name
# - Local users list
# - Shared folders
# - Password policies
# - Group memberships
```

### Phase 2 — Null Session Attack

Some misconfigured SMB servers allow **unauthenticated access**:

```bash
# Try null session (no username, no password)
smbclient -L //192.168.1.50 -N

# If successful — you see all shares:
# Sharename    Type    Comment
# ---------    ----    -------
# ADMIN$       Disk    Remote Admin
# C$           Disk    Default share
# IPC$         IPC     Remote IPC
# Finance      Disk    Finance Department Files  ← Interesting!
# HR_Data      Disk    Human Resources           ← Very interesting!
```

### Phase 3 — Accessing Shares

```bash
# Connect to an interesting share
smbclient //192.168.1.50/Finance -N

# Browse and download files
smb: \> ls
smb: \> get salary_2024.xlsx
smb: \> get passwords_backup.txt
```

### Phase 4 — NTLM Relay Attack

This is where SMB becomes extremely dangerous in real operations:

```
Step 1: Attacker sets up relay server (responder + ntlmrelayx)
    ↓
Step 2: Victim PC tries to authenticate to a network share
    ↓
Step 3: Attacker intercepts the NTLM authentication
    ↓
Step 4: Relays the authentication to ANOTHER machine
    ↓
Step 5: Gains access to that machine without ever cracking a password
```

```bash
# Tool: ntlmrelayx (Impacket)
ntlmrelayx.py -tf targets.txt -smb2support
```

### Common SMB Vulnerabilities

| Vulnerability | CVE | Impact |
|--------------|-----|--------|
| EternalBlue | MS17-010 | Remote Code Execution (WannaCry used this) |
| PrintNightmare | CVE-2021-1675 | Privilege Escalation to SYSTEM |
| SMB Null Session | Config issue | Unauthenticated enumeration |
| NTLM Relay | Config issue | Lateral movement without credentials |

---

## Defense Perspective

| Defense | How It Helps |
|---------|-------------|
| Disable SMBv1 | Removes EternalBlue attack surface |
| Enable SMB Signing | Prevents NTLM relay attacks |
| Block TCP 445 at perimeter | No external SMB access |
| Require authentication | Eliminate null sessions |

---

## Key Takeaways

- SMB = Windows file/resource sharing protocol
- Ports: **TCP 139** (legacy), **TCP 445** (modern)
- SMBv1 = extremely dangerous (EternalBlue)
- Tools: `enum4linux`, `smbclient`, `nmap SMB scripts`
- Null sessions = unauthenticated enumeration
- NTLM Relay = lateral movement without cracking passwords
- In real operations: SMB is the **primary lateral movement highway**

---
