# 05 — LDAP (Lightweight Directory Access Protocol)

> **Module:** Network Enumeration
> **Difficulty:** Beginner-Intermediate
> **RTO Relevance:** Critical — Direct pipeline to Active Directory data

---

## What is LDAP?

LDAP is a protocol used to **access and query directory services** — most importantly, **Active Directory**. It is the language used to communicate with AD to retrieve information about users, groups, computers, and policies.

Think of LDAP as a **search engine for Active Directory** — you send queries, AD returns results.

---

## How LDAP Works

```
Client (Attacker/Admin)          LDAP Server (Domain Controller)
        │                                    │
        │── "Give me all users in           │
        │    the Sales department" ────────→ │
        │                                    │
        │ ←── Returns list of users ──────── │
        │     with their attributes          │
```

LDAP organizes data in a **tree structure** called a Directory Information Tree (DIT):

```
dc=techcorp,dc=local          ← Root of the domain
├── ou=Users
│   ├── cn=ali.hassan         ← User object
│   └── cn=sara.khan
├── ou=Computers
│   └── cn=PC-KHI-01
└── ou=Groups
    └── cn=Domain Admins
```

---

## Important Ports

| Port | Protocol | Usage |
|------|----------|-------|
| TCP 389 | LDAP | Standard (unencrypted) |
| TCP 636 | LDAPS | LDAP over SSL (encrypted) |
| TCP 3268 | Global Catalog | Forest-wide queries |
| TCP 3269 | Global Catalog SSL | Encrypted forest-wide queries |

---

## LDAP Key Terms

| Term | Meaning | Example |
|------|---------|---------|
| dc | Domain Component | `dc=techcorp,dc=local` |
| ou | Organizational Unit | `ou=Sales` |
| cn | Common Name | `cn=ali.hassan` |
| DN | Distinguished Name | Full path to an object |
| Filter | Search query | `(objectClass=user)` |

---

## Red Team Perspective

### Why LDAP is a Goldmine

LDAP queries look like **normal network traffic** to defenders. A Red Teamer can quietly extract massive amounts of AD data without triggering alerts — it looks exactly like an admin doing routine queries.

### Anonymous/Unauthenticated LDAP Bind

Some misconfigured servers allow queries **without any credentials**:

```bash
# Test for anonymous LDAP bind
ldapsearch -H ldap://192.168.1.10 -x -b "dc=techcorp,dc=local"

# -H = host (LDAP server address)
# -x = simple authentication (anonymous)
# -b = base DN (where to start searching)
```

### Authenticated LDAP Enumeration

With valid credentials (even a low-privilege user), you can extract enormous amounts of data:

```bash
# Get ALL users in the domain
ldapsearch -H ldap://192.168.1.10 \
  -x -D "ali.hassan@techcorp.local" \
  -w "Password123" \
  -b "dc=techcorp,dc=local" \
  "(objectClass=user)"

# Get ALL groups
ldapsearch -H ldap://192.168.1.10 \
  -x -D "ali.hassan@techcorp.local" \
  -w "Password123" \
  -b "dc=techcorp,dc=local" \
  "(objectClass=group)"

# Find Domain Admins specifically
ldapsearch -H ldap://192.168.1.10 \
  -x -D "ali.hassan@techcorp.local" \
  -w "Password123" \
  -b "dc=techcorp,dc=local" \
  "(&(objectClass=user)(memberOf=CN=Domain Admins,CN=Users,DC=techcorp,DC=local))"
```

### Hunting for Kerberoastable Accounts via LDAP

```bash
# Find accounts with Service Principal Names (SPNs) set
# These are Kerberoasting targets
ldapsearch -H ldap://192.168.1.10 \
  -x -D "ali.hassan@techcorp.local" \
  -w "Password123" \
  -b "dc=techcorp,dc=local" \
  "(&(objectClass=user)(servicePrincipalName=*))" \
  sAMAccountName servicePrincipalName

# Output reveals:
# sAMAccountName: svc.database
# servicePrincipalName: MSSQL/db.techcorp.local
# ← This account is Kerberoastable!
```

### What LDAP Reveals to an Attacker

```
One authenticated LDAP session reveals:
├── All domain users (with attributes)
├── Password policies (min length, complexity, lockout)
├── All groups and memberships
├── All computers in the domain
├── Service accounts with SPNs (Kerberoasting targets)
├── Accounts with pre-auth disabled (AS-REP Roasting)
├── Admin accounts and their last login times
└── Password last set dates (find old/stale accounts)
```

---

## Defense Perspective

| Defense | How It Helps |
|---------|-------------|
| Disable anonymous LDAP bind | Requires authentication for all queries |
| Implement LDAP signing | Prevents relay attacks |
| Use LDAPS (port 636) | Encrypts LDAP traffic |
| Monitor unusual LDAP queries | Detect bulk enumeration |

---

## Key Takeaways

- LDAP = Query language for Active Directory
- Ports: **TCP 389** (standard), **TCP 636** (encrypted), **TCP 3268** (global catalog)
- Tool: `ldapsearch` — query AD directly from Linux
- Anonymous bind = unauthenticated AD access (misconfiguration)
- Even low-privilege user can dump enormous AD data via LDAP
- LDAP traffic looks normal — hard to detect during enumeration
- Critical use: Finding Kerberoastable accounts via SPN queries

---
