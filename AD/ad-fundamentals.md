# 01 — Active Directory Fundamentals

> **Module:** Active Directory
> **Difficulty:** Beginner-Intermediate
> **RTO Relevance:** Critical — Understanding AD architecture is the foundation of every internal attack

---

## What is Active Directory?

Active Directory (AD) is Microsoft's **centralized identity and access management system**. It acts as the brain of a Windows enterprise network — controlling who can log in, what they can access, and how every computer behaves.

Think of it like a large office building:
- Every employee has an **ID card** (user account)
- A **security guard** (Domain Controller) checks every ID at the door
- A **register** (AD database) holds every employee's record and permissions
- **Rules** (Group Policies) define what each employee can do

> **RTO Reality:** In real corporate environments, compromising Active Directory = compromising the entire organization. This is why understanding AD is non-negotiable for Red Team Operators.

---

## AD Architecture — The Full Picture

```
FOREST (Security Boundary — The entire empire)
│
├── TREE 1: techcorp.local (Root Domain — HQ)
│   │
│   ├── ROOT DOMAIN: techcorp.local
│   │   ├── DC: dc01.techcorp.local    ← Domain Controller
│   │   ├── OU: Executive Team
│   │   │   └── User: ceo.techcorp    ← High value target
│   │   └── OU: IT Admins
│   │       └── User: domain.admin    ← Ultimate target
│   │
│   ├── CHILD DOMAIN: asia.techcorp.local
│   │   ├── DC: dc02.asia.techcorp.local
│   │   ├── OU: Karachi Office
│   │   │   ├── User: ali.hassan      ← RTO starting point
│   │   │   └── Computer: PC-KHI-01
│   │   └── OU: Lahore Office
│   │       └── User: sara.khan
│   │
│   └── CHILD DOMAIN: europe.techcorp.local
│       └── OU: London Office
│           └── User: james.bond
│
└── TREE 2: techcorp.net (Separate business — Different name)
    └── ROOT DOMAIN: techcorp.net
        └── OU: Media Team
            └── User: editor.media
```

---

## Forest, Tree & Domain — Explained

### Forest
The **top-level security boundary** of Active Directory. Everything inside one forest shares:
- A common **schema** (blueprint defining what objects look like)
- A common **global catalog** (searchable index of all objects)
- **Trust relationships** between all domains

> One company = typically one forest.

### Tree
A collection of domains that share a **contiguous namespace** (connected naming structure).

```
techcorp.local              ← Tree root
├── asia.techcorp.local     ← Child (techcorp.local is IN the name)
└── europe.techcorp.local   ← Child (techcorp.local is IN the name)
```

### Child Domain vs New Tree

| Scenario | Type | Example |
|----------|------|---------|
| Domain name contains parent | Child Domain | `asia.techcorp.local` |
| Completely different name | New Tree | `techcorp.net` |

**Quick Rule:**
> Parent name visible in domain name → **Child Domain**
> Brand new different name → **New Tree**

### Real-World Examples

```
amazon.com          ← Forest Root Tree
aws.amazon.com      ← Child Domain (amazon.com is inside)
amazon.co.uk        ← New Tree (completely different name)

apple.com           ← Forest Root Tree
store.apple.com     ← Child Domain
music.apple.com     ← Child Domain
apple.co.uk         ← New Tree
```

---

## Organizational Units (OUs)

OUs are **folders inside a domain** used to organize objects (users, computers, groups) logically.

```
asia.techcorp.local
│
├── OU: Karachi Office
│   ├── User: ali.hassan
│   ├── User: sara.khan
│   └── Computer: PC-KHI-01
│
├── OU: Lahore Office
│   ├── User: ahmed.raza
│   └── Computer: PC-LHE-05
│
└── OU: IT Department
    ├── User: umar.admin
    └── Computer: SERVER-01
```

**Key Points:**
- OUs are for **organization only** — they are not security boundaries
- GPOs are applied at the OU level
- OUs can be **nested** (OUs inside OUs)

---

## Group Policy Objects (GPOs)

GPOs are **automated rule sets** that the Domain Controller pushes to users and computers. Write the rule once — it applies everywhere automatically.

### GPO Scope — Where They Apply

```
1. Local Policy       → Only one specific computer
        ↓
2. Site Policy        → Entire geographic location
        ↓
3. Domain Policy      → All objects in the domain
        ↓
4. OU Policy          → Specific OU only (most specific)
```

> **Priority Rule:** Lower (more specific) GPOs **override** higher ones.
> OU GPO wins over Domain GPO.

### GPO — Two Sections

**Computer Configuration**
Applies to the machine — regardless of who logs in:
```
- Disable USB ports
- Enable firewall
- Auto-install software
- Windows Update settings
```

**User Configuration**
Applies to the user — regardless of which machine they use:
```
- Set desktop wallpaper
- Block Control Panel access
- Run login/logout scripts
- Restrict specific applications
```

### RTO Perspective — GPO as a Weapon

```
Attacker gains Domain Admin access
        ↓
Creates new GPO named "Windows Update Policy"
(innocent-looking name)
        ↓
Adds malicious payload inside
        ↓
Links GPO to entire domain
        ↓
Every computer in domain executes payload
on next Group Policy refresh (90 minutes)
        ↓
5,000 machines compromised — one GPO
```

---

## Domain Controller (DC)

The DC is the **most critical server** in any AD environment. It runs Active Directory Domain Services (AD DS) and handles:

| Function | What It Does |
|----------|-------------|
| Authentication | Verifies every login request |
| Authorization | Decides what each user can access |
| Replication | Syncs data with other DCs |
| GPO Distribution | Pushes Group Policies to all machines |
| DNS | Resolves domain names internally |
| Time Sync | Keeps all machines on same time (critical for Kerberos) |

> **RTO Reality:** Domain Controller = The Crown. Every attack path in AD ultimately leads here.

---

## AD Objects

Everything in Active Directory is an **object**:

| Object Type | What It Is | RTO Relevance |
|-------------|-----------|---------------|
| User | Employee/service account | Credential target |
| Computer | Domain-joined machine | Lateral movement |
| Group | Collection of users/computers | Privilege escalation |
| GPO | Policy rule set | Mass payload delivery |
| OU | Organizational folder | Structure understanding |
| Domain | Network boundary | Attack scope |

---

## Trust Relationships — The Path Between Domains

Trusts allow users in one domain to access resources in another domain.

```
Child Domain: asia.techcorp.local
        ↓ (Trust relationship exists)
Parent Domain: techcorp.local
```

### RTO Trust Abuse

```
Attacker compromises asia.techcorp.local
        ↓
Uses trust relationship
        ↓
SID History Attack — claims Enterprise Admin rights
        ↓
Moves into techcorp.local (root domain)
        ↓
Entire Forest compromised
```

---

## Key Takeaways

- AD = Centralized identity management for Windows enterprise networks
- DC = Most critical server — ultimate RTO target
- Forest = Top security boundary (one per company, typically)
- Tree = Domains sharing contiguous namespace
- Child Domain = Parent name is part of its own name
- OU = Organizing folder within a domain (not a security boundary)
- GPO = Automated rules pushed by DC to users and computers
- Trust = Bridge between domains — abusable for forest-wide compromise

---
