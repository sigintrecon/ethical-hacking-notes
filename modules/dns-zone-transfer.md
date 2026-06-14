# 04 — DNS & Zone Transfer

> **Module:** Network Enumeration
> **Difficulty:** Beginner
> **RTO Relevance:** Medium — Reveals hidden subdomains and internal infrastructure map

---

## What is DNS?

DNS (Domain Name System) is the internet's phone book. It translates human-readable domain names into IP addresses that computers understand.

```
You type:     google.com
DNS returns:  142.250.185.46
Browser connects to that IP
```

---

## DNS Record Types

| Record | Purpose | RTO Value |
|--------|---------|-----------|
| A | Domain → IPv4 address | Target IP discovery |
| AAAA | Domain → IPv6 address | IPv6 target discovery |
| MX | Mail server for domain | Phishing infrastructure |
| NS | Name server for domain | Zone transfer target |
| TXT | Text records (SPF, DKIM) | Email security bypass intel |
| CNAME | Alias for another domain | Subdomain discovery |
| PTR | IP → Domain (reverse) | Internal hostname discovery |

---

## Important Ports

| Port | Protocol | Usage |
|------|----------|-------|
| UDP 53 | DNS | Standard queries (small data) |
| TCP 53 | DNS | Zone transfers (large data) |

> **Why two protocols?** UDP for quick single queries, TCP for bulk data transfers like zone transfers.

---

## DNS Enumeration with `dig`

### Finding Name Servers

```bash
# Find who manages the domain's DNS
dig target.com NS

# Short clean output
dig target.com NS +short

# Output example:
# ns1.hostingprovider.com
# ns2.hostingprovider.com
```

### Querying Specific Record Types

```bash
# A record — get IP address
dig target.com A

# MX record — find mail servers
dig target.com MX

# TXT record — find SPF, DKIM, verification codes
dig target.com TXT

# Find all records at once
dig target.com ANY
```

### Finding DNS Servers via WHOIS

```bash
# WHOIS often reveals Name Servers
whois target.com | grep -i "Name Server"

# The | grep -i "Name Server" part:
# |  = pipe operator (sends output to next command)
# grep = search tool
# -i = case-insensitive search
# "Name Server" = what we're searching for
```

---

## Zone Transfer Attack

### What is a Zone File?

A Zone File is a **complete database of all DNS records** for a domain — every subdomain, every internal server, every IP address.

```
Zone File for techcorp.com:
├── techcorp.com              → 203.0.113.10  (Main website)
├── mail.techcorp.com         → 203.0.113.11  (Mail server)
├── vpn.techcorp.com          → 203.0.113.12  (VPN gateway)
├── admin.techcorp.com        → 192.168.1.5   (Internal admin panel!)
├── dev-staging.techcorp.com  → 192.168.1.20  (Dev server!)
└── secret-db.techcorp.com    → 192.168.1.100 (Database server!)
```

### Why Does Zone Transfer Exist?

Companies have multiple DNS servers (primary + backup). The backup needs to sync its records from the primary — this sync process is called a **Zone Transfer (AXFR)**.

```
Primary DNS Server                Secondary DNS Server
(Has full zone file)              (Needs a copy for backup)
        │                                  │
        │ ←── "Give me your zone file" ──── │
        │                                  │
        │ ──── Sends entire zone file ────→ │
        │         (AXFR transfer)           │
```

### The Attack — Misconfigured Zone Transfer

If the DNS server is not configured to **restrict zone transfers** to only trusted IPs, ANY computer can request the entire zone file:

```bash
# Step 1: Find the name server
dig target.com NS +short
# ns1.target.com

# Step 2: Request zone transfer from that name server
dig axfr @ns1.target.com target.com

# If misconfigured — you get the ENTIRE zone file:
# target.com.        SOA   ns1.target.com. ...
# mail.target.com.   A     203.0.113.11
# vpn.target.com.    A     203.0.113.12
# admin.target.com.  A     192.168.1.5      ← Found internal admin!
# db.target.com.     A     192.168.1.100    ← Found database!
```

### Practice Target (Legal)

```bash
# zonetransfer.me is a legal practice target
dig axfr @nsztm1.digi.ninja zonetransfer.me
```

### RTO Value of Zone Transfer Data

```
Zone Transfer Successful
    ↓
Complete infrastructure map obtained:
├── All public-facing servers (web, mail, VPN)
├── Internal server names and IPs
├── Development/staging environments
├── Admin panels and management interfaces
└── Database servers
    ↓
Attack planning becomes surgical, not guesswork
```

### DNS as a C2 Channel (Advanced RTO)

DNS is almost **never blocked** by firewalls (port 53 must stay open for internet to work). Red Teamers exploit this:

```
Compromised machine inside company
    ↓
All outbound ports blocked by firewall
    ↓
But DNS (UDP 53) is always open
    ↓
Attacker encodes commands inside DNS queries
    ↓
Data exfiltrated through DNS — completely bypasses firewall
```

This technique is called **DNS Tunneling**.

---

## Defense Perspective

| Defense | How It Helps |
|---------|-------------|
| Restrict zone transfers to specific IPs | Only backup DNS servers can request zone files |
| Split-horizon DNS | Internal and external DNS return different records |
| Monitor DNS queries | Detect DNS tunneling via unusual query patterns |
| Rate-limit DNS responses | Prevent automated enumeration |

---

## Key Takeaways

- DNS = Translates domain names to IP addresses
- Key tool: `dig` — query any DNS record type
- Zone File = Complete map of all DNS records for a domain
- Zone Transfer (AXFR) = Backup sync process — **often misconfigured**
- Misconfigured AXFR = Entire infrastructure map leaked in one command
- DNS also used for C2 communication via **DNS Tunneling**
- Port: UDP 53 (queries), TCP 53 (zone transfers)

---

*← [SMB](03-smb.md) | Back to [README](../../README.md) | Next: [LDAP →](05-ldap.md)*
