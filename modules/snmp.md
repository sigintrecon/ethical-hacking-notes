# 02 — SNMP (Simple Network Management Protocol)

> **Module:** Network Enumeration
> **Difficulty:** Beginner
> **RTO Relevance:** Medium-High — Misconfigured SNMP leaks entire network topology

---

## What is SNMP?

SNMP is a protocol used to **monitor and manage network devices** from a central location. It allows network administrators to check the health, performance, and configuration of routers, switches, servers, and printers — all from one place.

Think of it like a hospital monitoring system — every patient (device) has sensors (SNMP agent) reporting their vitals to the central nurse station (SNMP manager).

---

## How SNMP Works — Client-Server Model

SNMP operates on a **Manager-Agent** (Client-Server) model:

```
SNMP Manager (Client)          SNMP Agent (Server)
[Nagios / SolarWinds]    ←→    [Router / Switch / Server]
    "Give me your data"              "Here is my data"
         Port UDP 161 →
         ← Response UDP 161
         ← Trap Alert UDP 162
```

### Components

**SNMP Manager**
The central monitoring software (Nagios, SolarWinds, PRTG) that sends queries to devices and collects data.

**SNMP Agent**
A small program built into every managed device. It sits on **UDP 161**, waits for queries, and responds with device data.

**MIB (Management Information Base)**
A database structure that defines what data an agent can provide. Think of it as a menu — the manager orders from this menu.

---

## Important Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| UDP 161 | SNMP | Manager queries Agent (Request/Response) |
| UDP 162 | SNMP Trap | Agent alerts Manager (Device sends alert automatically) |

---

## Community Strings — The Critical Weakness

Community strings are essentially **passwords** for SNMP access. There are two types:

| Type | Default Value | Access Level |
|------|--------------|--------------|
| Read-Only (RO) | `public` | Read device data only |
| Read-Write (RW) | `private` | Read AND modify device config |

**The Problem:** Many admins never change these defaults.

---

## Red Team Perspective

### Attack 1 — SNMP Enumeration (Default Community Strings)

If the community string `public` is still set, an attacker can dump the entire device configuration:

```bash
# Enumerate ALL SNMP data from a target
snmpwalk -v2c -c public 192.168.1.1

# What you get back:
# System info (OS, hostname, uptime)
# Network interfaces and IP addresses
# Running processes
# Installed software
# Routing tables
# Connected devices list
```

This single command can reveal the **entire network topology** without any authentication.

### Attack 2 — Targeted SNMP Queries

```bash
# Get system description
snmpget -v2c -c public 192.168.1.1 1.3.6.1.2.1.1.1.0

# Get list of running processes
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.25.4.2

# Get network interfaces
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.2.2
```

### Attack 3 — Brute Force Community Strings

```bash
# Using Nmap NSE script
nmap -sU -p 161 --script snmp-brute 192.168.1.1

# Using onesixtyone
onesixtyone -c community-strings.txt 192.168.1.1
```

### What an Attacker Gains from SNMP

```
Default community string found
    ↓
snmpwalk dumps entire device data
    ↓
Attacker learns:
├── All IP addresses in the network
├── All connected devices (routers, switches, servers)
├── Running services on each device
├── Network routing paths
└── Device OS versions (for exploit selection)
    ↓
Complete network map — without touching a single machine
```

### SNMP v1/v2c vs v3

| Version | Authentication | Encryption | Security |
|---------|---------------|------------|---------|
| v1 | Community String | None | ❌ Very Weak |
| v2c | Community String | None | ❌ Weak |
| v3 | Username + Password | Yes (AES/DES) | ✅ Secure |

> **RTO Note:** If you see SNMP v1 or v2c in a network scan — it is almost always worth investigating. v3 is significantly harder to abuse.

---

## Defense Perspective

| Defense | How It Helps |
|---------|-------------|
| Change default community strings | Eliminates trivial access |
| Use SNMPv3 | Adds authentication and encryption |
| Firewall UDP 161/162 | Block external SNMP access |
| Whitelist manager IPs | Only allow specific hosts to query |

---

## Key Takeaways

- SNMP = Network monitoring protocol (Manager-Agent model)
- Ports: **UDP 161** (queries), **UDP 162** (traps/alerts)
- Community strings = passwords (`public` = read, `private` = write)
- Default community strings = **instant network topology leak**
- Tool: `snmpwalk` — dumps all device data
- v1/v2c = weak (no encryption), v3 = secure

---
