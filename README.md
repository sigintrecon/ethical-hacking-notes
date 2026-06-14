# Ethical Hacking & Red Team Notes

> **A structured, beginner-to-advanced knowledge base for aspiring Red Team Operators and Penetration Testers — built from real-world concepts, not just theory.**

[![GitHub](https://img.shields.io/badge/Level-Beginner%20to%20Advanced-blue)]()
[![Focus](https://img.shields.io/badge/Focus-Red%20Team%20Operations-red)]()
[![Status](https://img.shields.io/badge/Status-Active%20%7C%20Continuously%20Updated-green)]()

---

## Legal Disclaimer

> **READ BEFORE PROCEEDING**
>
> This repository is created **strictly for educational purposes**. All concepts, techniques, and methodologies documented here are intended to help individuals learn ethical hacking, cybersecurity defense, and red team operations in **authorized, legal environments only**.
>
> - ✅ Use this knowledge in **CTF (Capture The Flag)** competitions
> - ✅ Use this in **your own lab environments** (VirtualBox, VMware)
> - ✅ Use this in **authorized penetration tests** with written permission
> - ❌ **Never** use these techniques on systems you do not own or have explicit written permission to test
> - ❌ Unauthorized access to computer systems is a **criminal offense** in every country
>
> The author takes **no responsibility** for any misuse of the information provided. You are solely responsible for your own actions.

---

## About This Repository

This repo documents a structured journey through **ethical hacking and red team operations** — from foundational networking concepts to advanced Active Directory attacks. Every concept is explained from a **Red Team Operator (RTO) perspective**, not just a theoretical angle.

**Who is this for?**
- 🟢 Beginners who want to understand how real attacks work
- 🟡 Intermediate learners preparing for certifications (CEH, OSCP, CRTO)
- 🔴 Advanced practitioners who want a quick reference guide

---

## Repository Structure

```
ethical-hacking-notes/
│
├── README.md                          ← You are here (Start here!)
│
└── modules/
    ├── 01-network-enumeration/
    │   ├── 01-netbios.md              ← NetBIOS Protocol
    │   ├── 02-snmp.md                 ← SNMP Protocol
    │   ├── 03-smb.md                  ← SMB Protocol
    │   ├── 04-dns-zone-transfer.md    ← DNS & Zone Transfer
    │   ├── 05-ldap.md                 ← LDAP Enumeration
    │   ├── 06-nfs-rpc.md              ← NFS & RPC
    │   └── 07-os-fingerprinting.md    ← OS Fingerprinting
    │
    └── 02-active-directory/
        ├── 01-ad-fundamentals.md      ← AD Core Concepts
        ├── 02-ad-authentication.md    ← NTLM & Kerberos
        ├── 03-ad-users-groups.md      ← Users, Groups & ACLs
        ├── 04-ntds-dit.md             ← NTDS.dit — Crown Jewel
        └── 05-ad-attack-paths.md      ← Attack Paths & Techniques
```

---

## Learning Roadmap

```
PHASE 1 — FOUNDATION
├── Understand how networks communicate
├── Learn enumeration protocols
└── Set up your lab environment

        ↓

PHASE 2 — ACTIVE DIRECTORY
├── Understand AD architecture
├── Learn authentication mechanisms
└── Study privilege levels & attack surfaces

        ↓

PHASE 3 — EXPLOITATION (Coming Soon)
├── Hands-on attacks in lab
├── BloodHound & PowerView usage
└── AD attack chains

        ↓

PHASE 4 — RED TEAM OPS (Coming Soon)
├── C2 Frameworks (Sliver, Havoc)
├── AV/EDR Evasion
└── Full Red Team Engagements
```

---

## Modules

### Module 1 — Network Enumeration

| # | Topic | Real-World Relevance | File |
|---|-------|---------------------|------|
| 01 | NetBIOS | LLMNR Poisoning, Hash Capture | [netbios.md](modules/01-network-enumeration/01-netbios.md) |
| 02 | SNMP | Network Device Recon, Config Dump | [snmp.md](modules/01-network-enumeration/02-snmp.md) |
| 03 | SMB | Lateral Movement, Share Enumeration | [smb.md](modules/01-network-enumeration/03-smb.md) |
| 04 | DNS Zone Transfer | Subdomain Discovery, Recon | [dns-zone-transfer.md](modules/01-network-enumeration/04-dns-zone-transfer.md) |
| 05 | LDAP | AD Enumeration, User Harvesting | [ldap.md](modules/01-network-enumeration/05-ldap.md) |
| 06 | NFS/RPC | File System Access, Info Disclosure | [nfs-rpc.md](modules/01-network-enumeration/06-nfs-rpc.md) |
| 07 | OS Fingerprinting | Target Profiling, Exploit Selection | [os-fingerprinting.md](modules/07-os-fingerprinting.md) |

### Module 2 — Active Directory

| # | Topic | Real-World Relevance | File |
|---|-------|---------------------|------|
| 01 | AD Fundamentals | Forest, Tree, Domain, OUs, GPOs | [ad-fundamentals.md](modules/02-active-directory/01-ad-fundamentals.md) |
| 02 | AD Authentication | NTLM, Kerberos, Ticket Attacks | [ad-authentication.md](modules/02-active-directory/02-ad-authentication.md) |
| 03 | Users, Groups & ACLs | Privilege Levels, ACL Abuse | [ad-users-groups.md](modules/02-active-directory/03-ad-users-groups.md) |
| 04 | NTDS.dit | Crown Jewel, DCSync, Dumping | [ntds-dit.md](modules/02-active-directory/04-ntds-dit.md) |
| 05 | AD Attack Paths | Full Attack Chain, RTO Mindset | [ad-attack-paths.md](modules/02-active-directory/05-ad-attack-paths.md) |

---

## Key Mindset — Pentester vs Red Teamer

| | Penetration Tester | Red Team Operator |
|--|-------------------|-------------------|
| **Goal** | Find vulnerabilities | Simulate real attacker |
| **Noise** | Acceptable | Must stay silent |
| **Tools** | Metasploit, Nmap freely | Custom payloads, LotL |
| **Blue Team** | Knows testing is happening | Has NO idea |
| **Report** | List of vulnerabilities | Full attack narrative |
| **Timeframe** | Days to weeks | Weeks to months |

---

## Lab Setup (Recommended)

```
Virtualization:  VirtualBox or VMware
Attacker OS:     Kali Linux
Target 1:        Windows Server 2019 (DC)
Target 2:        Windows 10 Pro (Domain joined)
Target 3:        Metasploitable2 (Linux targets)
Network:         Host-Only or Internal Network
```

---

## Recommended Resources

| Resource | Type | Level |
|----------|------|-------|
| [TryHackMe](https://tryhackme.com) | Interactive Labs | Beginner |
| [HackTheBox](https://hackthebox.com) | CTF / Labs | Intermediate |
| [PortSwigger Web Academy](https://portswigger.net/web-security) | Web Hacking | Beginner-Advanced |
| [TCM Security Courses](https://tcm-sec.com) | Video Courses | Beginner-Advanced |
| [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) | Cheat Sheets | All Levels |

---

## Quick Reference — Important Ports

| Port | Protocol | RTO Relevance |
|------|----------|---------------|
| UDP 137-138, TCP 139 | NetBIOS | LLMNR Poisoning |
| UDP 161-162 | SNMP | Device Recon |
| TCP 445 | SMB | Lateral Movement |
| TCP/UDP 53 | DNS | Zone Transfer, C2 Tunneling |
| TCP 389, 636 | LDAP | AD Enumeration |
| TCP 88 | Kerberos | Ticket Attacks |
| TCP 135 | RPC | Enumeration |
| TCP 3268-3269 | Global Catalog | Forest-wide AD Queries |

---

## Contributing

This is a personal learning repository. If you find errors or want to suggest improvements, feel free to open an issue.

---

## License

This repository is for **educational use only**. See [DISCLAIMER](#️-legal-disclaimer) above.

---

*"The quieter you become, the more you are able to hear." — Red Team Mindset*
