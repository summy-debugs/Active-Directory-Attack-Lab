# ACTIVE DIRECTORY ATTACK LAB REPORT 

## Executive Summary

This report documents a comprehensive Active Directory penetration test lab environment demonstrating a complete attack chain from initial compromise through full domain takeover. The lab simulates a realistic enterprise environment with a 
- Domain Controller (DC01), 
- workstation (WS01), 
- attacker machine (Kali Linux).
 
Through a series of well-coordinated attacks, complete domain compromise was achieved, resulting in SYSTEM-level access to the Domain Controller and extraction of all domain credentials.

**Key Findings:**
- Successfully exploited LLMNR poisoning vulnerability to capture user credentials
- Cracked captured hashes to obtain valid domain credentials
- Executed Kerberoasting attack against service accounts
- Performed DCSync attack to dump all domain credentials
- Achieved code execution as SYSTEM on the Domain Controller
- Created domain visualization using BloodHound

## Table of Contents

- [Executive Summary](#executive-summary)
- [Lab Architecture](#lab-architecture)
  - [Network Configuration](#network-configuration)
  - [Domain Structure](#domain-structure)
- [Attack Chain](#attack-chain)
  - [Phase 1: LLMNR Poisoning & Hash Capture](#phase-1-llmnr-poisoning--hash-capture)
  - [Phase 2: Hash Cracking](#phase-2-hash-cracking)
  - [Phase 3: Kerberoasting](#phase-3-kerberoasting)
  - [Phase 4: DCSync Attack](#phase-4-dcsync-attack)
  - [Phase 5: Pass-the-Hash & Remote Code Execution](#phase-5-pass-the-hash--remote-code-execution)
  - [Phase 6: BloodHound Enumeration](#phase-6-bloodhound-enumeration)
- [Tools Used](#tools-used)
- [Attack Timeline](#attack-timeline)
- [Attack Summary](#attack-summary)
- [MITRE ATT&CK Mapping](#mitre-attck-mapping)
- [Lessons Learned & Security Recommendations](#lessons-learned--security-recommendations)
- [Additional Security Controls](#additional-security-controls)
- [Conclusion](#conclusion)
- [Skills Demonstrated](#skills-demonstrated)
- [References](#references)
- [Appendix A – MITRE ATT&CK Mapping](#appendix-a--mitre-attck-mapping)
- [Appendix B – Lab Environment](#appendix-b--lab-environment)
- [Appendix C – Report Information](#appendix-c--report-information)


## Lab Architecture

### Network Configuration
- **Domain:** MARVEL.local
- **DC01 (Domain Controller):** 192.168.50.1 - Windows Server 2022
- **WS01 (Workstation):** 192.168.50.100 - Windows 11 Enterprise
- **Kali (Attacker):** 192.168.50.10 - Kali Linux

### Domain Structure
- **Users Created:** tstark - Tony Stark , pparker - Peter Parker, srogers - Steven Rogers , sqlservice
- **Groups:** Marvel Admins (Domain Admins)
- **Service Principal Names (SPNs):** MARVEL/SQLService.MARVEL.local:60111
- **Organizational Units:** Groups, Service Accounts


![Active Directory Groups](./screenshots/setup/groups.png)
![tstark Member Of](./screenshots/setup/tstark-member-of.png)
![tstark Properties](./screenshots/setup/tstark-properties.png)


## Attack Chain

### Phase 1: LLMNR Poisoning & Hash Capture

**Objective:** Capture NTLMv2 hashes from domain users through LLMNR poisoning

**LLMNR** (Link-Local Multicast Name Resolution)

- LLMNR is a Windows name resolution protocol that is used when DNS cannot resolve a hostname. It broadcasts a request on the local network asking if any device knows the requested host. Since any system can respond, an attacker running Responder can spoof the reply and trick the victim into connecting to the attacker's machine.

**NTLMv2** (NT LAN Manager Version 2)

- NTLMv2 is a Windows authentication protocol that verifies a user's identity without sending the actual password over the network. Instead, it sends a challenge-response hash. If this hash is captured by an attacker, it can be subjected to offline password cracking or used in other attacks if additional protections are absent.


**Tools Used:** Responder 
- Responder is a network penetration testing tool used to monitor and respond to LLMNR, NBT-NS, and mDNS name resolution requests. It is commonly used during Active Directory security assessments to demonstrate how misconfigured name resolution protocols can lead to NTLM authentication hash capture.

**Attack Steps:**

1. Started Responder on the Kali Linux attacker machine to listen for LLMNR, NBT-NS, and mDNS requests.
2. Verified that TCP port **445 (SMB)** was not occupied by another service, ensuring Responder could bind to the SMB listener.
3. Triggered an LLMNR broadcast from **WS01** by attempting to access a non-existent network share (`\\MARVELSEC`).
4. Responder intercepted the LLMNR request and replied with a spoofed response, causing the victim to attempt authentication.
5. WS01 authenticated to Responder over SMB, allowing the capture of the user's NTLMv2 authentication hash.

**Commands Used:**
```bash
# Start Responder on Kali
sudo responder -I eth0 -wv

# On WS01, trigger LLMNR request
\\MARVELSEC
```

#### Troubleshooting

During the initial attempt, Responder successfully intercepted and poisoned LLMNR requests but failed to capture SMB authentication hashes because the SMB service could not bind to port 445. This was caused by another process already using the port. The conflicting process was terminated, Responder was restarted, and the attack was repeated successfully, resulting in the capture of the NTLMv2 hash.

**Results:**
- Successfully captured tstark's NTLMv2 hash
- Hash format: `tstark::MARVEL:f7f58385b661c344:88A47BB16064969C0DF5B823FB7D599B:...`

![NTLMv2 Hash Captured from tstark](./screenshots/attack/llmnr/phase1-hash.png)


### Phase 2: Hash Cracking

**Objective:** Crack captured NTLMv2 hash offline

**Tools Used:** John the Ripper
- John the Ripper is a password-cracking tool that performs offline attacks against captured password hashes using wordlists, dictionaries, or brute-force techniques.

**Attack Steps:**
1. Saved captured hash to file
2. Executed John against fasttrack.txt wordlist which is available as default in usr/share/wordlists in Kali ( Attacker Machine )
3. Successfully cracked hash 

**Commands Used:**
```bash
# Save hash to file
echo "tstark::MARVEL:f7f58385b661c344:88A47BB16064969C0DF5B823FB7D599B:..." > /tmp/tstark.hash

# Crack with John
john /tmp/tstark.hash --wordlist=/usr/share/wordlists/fasttrack.txt
```

**Results:**
- **Username:** tstark
- **Password:** Password1!

![John the Ripper Cracked tstark Hash](./screenshots/attack/llmnr/phase2-john.png)


### Phase 3: Kerberoasting

#### Understanding Kerberoasting
- A service account is a special Windows user account used by applications or services (such as SQL Server, IIS, or backup software) to run automatically. Just like a normal user account, it has a username and password, but instead of a person logging in, a service uses the account.

Example:
- Username: sqlservice
- Password: MYpassword123#

#### What is an SPN (Service Principal Name)?

An SPN is a unique name that tells Active Directory which account is running a particular service.

Example:
MARVEL/SQLService.MARVEL.local:60111

This tells Active Directory that the SQL Server service belongs to the `sqlservice` account.

#### How Kerberos Normally Works

Suppose a user wants to connect to SQL Server.

1. The user asks the Domain Controller for permission to access the service.
2. The Domain Controller creates a Kerberos service ticket (called a TGS ticket).
3. The ticket is encrypted using the password of the service account.
4. The user presents this ticket to SQL Server, proving they have permission to access it.

At no point is the service account password sent across the network.

#### So where is the weakness?

The important thing is that **any authenticated domain user** can request a TGS ticket for a service account.

For example, after compromising the `tstark` account, we could simply ask the Domain Controller:

"Give me the ticket for the SQL service."

The Domain Controller assumes this is a legitimate request and returns the encrypted ticket.

#### Why can the password be cracked?

The TGS ticket is encrypted using the service account's password.

Although the password itself is never sent, the encrypted ticket can be saved and attacked offline.

Tools such as John the Ripper repeatedly try passwords from a wordlist until one successfully decrypts the ticket.

If a match is found, the correct service account password has been recovered.


**Objective:** Request and crack Kerberos TGS tickets for service accounts.

**Tools Used:** Impacket GetUserSPNs, John the Ripper

**Tool Descriptions:**
- **Impacket GetUserSPNs:** Enumerates Service Principal Names (SPNs) and requests Kerberos TGS tickets for service accounts.
- **John the Ripper:** Performs offline password cracking against captured Kerberos tickets.

**Attack Steps:**
1. Used the compromised `tstark` credentials to authenticate to the domain.
2. Enumerated Service Principal Names (SPNs) using GetUserSPNs.
3. Identified the `sqlservice` account with the SPN `MARVEL/SQLService.MARVEL.local:60111`.
4. Requested the Kerberos TGS ticket for the service account and saved it to a file.
5. Used John the Ripper with the `rockyou.txt` wordlist to crack the captured TGS ticket.
6. Successfully recovered the service account password.

**Commands Used:**
```bash
# Request and save Kerberos TGS ticket
python3 /usr/share/doc/python3-impacket/examples/GetUserSPNs.py -request -dc-ip 192.168.50.1 MARVEL.local/tstark:Password1! > /tmp/sqlservice.ticket

# Crack Kerberos ticket
john --format=krb5tgs /tmp/sqlservice.ticket --wordlist=/usr/share/wordlists/rockyou.txt
```

**Results:**
- **Service Account:** `sqlservice`
- **Password:** `MYpassword123#`
- Demonstrates that authenticated users can request service tickets for SPN-enabled accounts and crack weak service account passwords offline.

![SPN Configuration](./screenshots/setup/spn-config.png)
![GetUserSPNs Output](./screenshots/attack/kerberoasting/phase3-spns.png)
![John Cracked sqlservice Ticket](./screenshots/attack/kerberoasting/phase3-john.png)




### Phase 4: DCSync Attack

#### Understanding DCSync 


#### What is a Domain Controller?

A Domain Controller (DC) is the main server in an Active Directory environment. It stores important information about the domain, including:

- User accounts
- Password hashes
- Computer accounts
- Group memberships
- Security policies

Whenever a user logs in, changes a password, or accesses domain resources, the Domain Controller verifies the request.

#### Why do Domain Controllers replicate?

Large organizations often have multiple Domain Controllers.

For example:

DC01
DC02

If a user changes their password on DC01, DC02 must also receive the updated information. To keep all Domain Controllers synchronized, Windows automatically performs **replication**, where Domain Controllers exchange Active Directory data.

#### What is DCSync?

DCSync is an attack that abuses the Active Directory replication process.

Instead of stealing the Active Directory database directly, the attacker pretends to be another Domain Controller and requests replication data.

If the attacker has the required replication privileges (such as Domain Admin rights), the real Domain Controller trusts the request and sends Active Directory data, including password hashes.

#### Why is DCSync dangerous?

Unlike many attacks, DCSync does not require the attacker to log in to the Domain Controller or copy the NTDS.dit database.

Instead, it uses the normal Active Directory replication process to retrieve sensitive information.

This makes the attack quieter and more difficult to detect than directly accessing the database.

#### What is secretsdump.py?

`secretsdump.py` is an Impacket tool used to extract Windows credentials.

During a DCSync attack, it sends a replication request to the Domain Controller and retrieves:

- NTLM password hashes
- Kerberos keys
- Password history (if available)

These credentials can then be used for additional attacks such as Pass-the-Hash or Golden Ticket attacks.

#### DCSync in this Lab

1. The `tstark` account was added to the **Domain Admins** group.
2. Using these elevated privileges, `secretsdump.py` was executed from the Kali machine.
3. The tool requested Active Directory replication from the Domain Controller.
4. The Domain Controller treated the request as legitimate and returned password hashes for all domain users.
5. The extracted hashes demonstrated complete compromise of the Active Directory environment.

**Objective:** Extract password hashes of domain users by impersonating a Domain Controller.

**Tools Used:** Impacket secretsdump

**Tool Description:**
- **Impacket secretsdump:** Extracts credentials from Windows systems. When used with DCSync, it requests password hashes directly from the Domain Controller using the Directory Replication Service (DRS).

**Attack Steps:**
1. Added the `tstark` account to the **Domain Admins** group to obtain replication privileges.
2. Executed the DCSync attack against the Domain Controller using `secretsdump.py`.
3. The Domain Controller treated the attacker as a replication partner and replicated password hashes.
4. Retrieved NTLM hashes and Kerberos keys for domain users.

**Results:**
- Successfully extracted NTLM password hashes.
- Retrieved Kerberos keys for domain accounts.
- Demonstrated complete compromise of Active Directory credentials.

![tstark Added to Domain Admins](./screenshots/setup/phase4-domainadmin1.png)
![secretsdump DCSync Output](./screenshots/attack/dcsync/phase4-dcsync.png)

**Commands Used:**
```bash
# Add tstark to Domain Admins
net group "Domain Admins" tstark /add /domain

# Execute DCSync
python3 /usr/share/doc/python3-impacket/examples/secretsdump.py MARVEL.local/tstark:Password1!@192.168.50.1
```
**Impact:** Complete domain compromise - krbtgt hash enables creation of forged Kerberos tickets (Golden Tickets)

[INSERT SCREENSHOT: secretsdump output showing all domain hashes]
[INSERT SCREENSHOT: krbtgt hash highlighted]


### Phase 5: Pass-the-Hash & Remote Code Execution

#### Understanding Pass-the-Hash (Beginner Friendly)

Before understanding Pass-the-Hash, it is important to know what an NTLM hash is.

## What is an NTLM Hash?

When a Windows user creates a password, Windows does not store the actual password. Instead, it stores a mathematical representation of the password called an **NTLM hash**.

For example:

Password:
Password1!

Stored in Active Directory as:

NTLM Hash:
7facdc498ed1680c4fd1448319a8c04f

This means that even if someone gains access to the Active Directory database, they usually obtain password hashes rather than the plaintext passwords.



## How does Windows authenticate?

Normally, when a user logs in:

1. The user enters their password.
2. Windows converts the password into an NTLM hash.
3. The generated hash is compared with the hash stored in Active Directory.
4. If both hashes match, the user is authenticated.

Notice that Windows is comparing **hashes**, not the actual password.


#### What is Pass-the-Hash?

Pass-the-Hash (PtH) is an attack where an attacker authenticates using an NTLM hash instead of the user's plaintext password.

Since Windows accepts the NTLM hash during authentication, the attacker does not need to know the original password.

If the correct NTLM hash is available, it can be "passed" directly to Windows, which treats it as a valid authentication credential.


#### Where did the Administrator hash come from?

Earlier in the lab, the DCSync attack was performed using `secretsdump.py`.

The Domain Controller replicated Active Directory credentials, including the Administrator's NTLM hash.

Example:

Administrator
↓

NTLM Hash:
7facdc498ed1680c4fd1448319a8c04f

Although the Administrator's password was never recovered, the hash itself was enough to authenticate.


#### What does psexec.py do?

`psexec.py` is an Impacket tool that remotely executes commands on Windows systems.

When used with the Pass-the-Hash technique, it:

1. Uses the supplied NTLM hash to authenticate.
2. Connects to the target system over SMB.
3. Creates a temporary Windows service.
4. Starts the service.
5. Opens an interactive command prompt for the attacker.

If the supplied hash belongs to an Administrator account, the resulting shell runs as:

NT AUTHORITY\SYSTEM

This is the highest privilege level on a Windows computer.



#### Why is SYSTEM important?

The SYSTEM account has more privileges than a normal Administrator.

A SYSTEM shell allows an attacker to:

- Execute any command.
- Read and modify protected files.
- Create or delete user accounts.
- Install software or malware.
- Disable security controls.
- Take complete control of the machine.

Obtaining a SYSTEM shell on a Domain Controller effectively means the attacker has full control over the Active Directory environment.

#### Pass-the-Hash in this Lab

1. The DCSync attack extracted the Administrator NTLM hash.
2. The Administrator password was not required.
3. `psexec.py` used the NTLM hash to authenticate to DC01.
4. Authentication succeeded because the hash was valid.
5. A temporary service was created on the Domain Controller.
6. An interactive command shell running as **NT AUTHORITY\SYSTEM** was obtained.
7. The `whoami` command confirmed SYSTEM-level privileges.

This demonstrates that possession of an NTLM hash can be as powerful as knowing the user's password, highlighting the importance of protecting credential hashes within Active Directory.

**Objective:** Authenticate to the Domain Controller using an NTLM hash and obtain a SYSTEM-level interactive shell.


**Tools Used:** Impacket psexec

**Tool Description:**
- **Impacket psexec:** A remote administration tool that executes commands on Windows systems. It supports the Pass-the-Hash technique, allowing authentication using an NTLM hash instead of the plaintext password.

***Attack Steps***

1. Used the Administrator NTLM hash obtained during the DCSync attack.
2. Executed `psexec.py` from the Kali attacker machine using the Pass-the-Hash technique.
3. Authenticated to the Domain Controller without knowing the Administrator's plaintext password.
4. Uploaded the required service executable and created a temporary Windows service.
5. The service executed on the Domain Controller and spawned an interactive command shell.
6. Verified SYSTEM-level privileges using the `whoami` command.

***Results***

- Successfully authenticated using only the NTLM hash.
- Obtained an interactive shell on **DC01**.
- Confirmed execution as **NT AUTHORITY\SYSTEM**.
- Successfully executed Windows commands such as `whoami` and `ipconfig`.
- Demonstrated complete administrative control over the Domain Controller.

### Commands Used

```bash
# Pass-the-Hash using Administrator NTLM hash
python3 /usr/share/doc/python3-impacket/examples/psexec.py \
-hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f \
Administrator@192.168.50.1

# Verify SYSTEM privileges
whoami
ipconfig
```


![NT AUTHORITY\SYSTEM Shell on DC01](./screenshots/attack/pass-the-hash/phase5-system.png)



### Phase 6: BloodHound Domain Analysis

**Objective:** Enumerate the Active Directory environment and visualize domain relationships, permissions, and potential attack paths.

**Tools Used:** BloodHound, bloodhound-python, Neo4j

***Tool Descriptions:***
- **BloodHound:** Visualizes Active Directory objects and relationships using an interactive graph.
- **bloodhound-python:** Collects Active Directory information and exports it as JSON files for BloodHound.
- **Neo4j:** Graph database used by BloodHound to store and display collected data.

***Attack Steps***

1. Installed and configured Neo4j as the backend database.
2. Started the Neo4j service and launched the BloodHound application.
3. Used the compromised `tstark` credentials to collect Active Directory information with `bloodhound-python`.
4. Generated JSON files containing domain users, groups, computers, sessions, Organizational Units (OUs), ACLs, and trust relationships.
5. Imported the generated JSON files into the BloodHound GUI.
6. Analyzed the resulting graph to identify relationships and potential privilege escalation paths.


***Results***

- Successfully enumerated the MARVEL Active Directory domain.
- Visualized the relationship between `tstark` and the **Domain Admins** group.
- Identified the `sqlservice` service account and its associated SPN.
- Displayed relationships between users, groups, computers, and permissions.
- Produced a complete graphical representation of the Active Directory environment.

#### Commands Used

```bash
# Start Neo4j
sudo neo4j start

# Launch BloodHound
bloodhound

# Collect Active Directory data
bloodhound-python -u tstark -p Password1! -ns 192.168.50.1 -d MARVEL.local -c All
```

After data collection:

1. Open BloodHound.
2. Log in using the Neo4j credentials.
3. Click **Upload Data**.
4. Import all generated JSON files.
5. Analyze the generated graph.

### Evidence

![BloodHound MARVEL.local Domain Graph](./screenshots/attack/bloodhound/phase6-bloodhound.png)
![BloodHound Attack Path Visualization](./screenshots/attack/bloodhound/phase6-bloodhound1.png)


## Attack Chain Overview

The following diagram summarizes the complete attack sequence performed during this Active Directory penetration testing lab.

```text
                     Initial Access
                           │
                           ▼
              LLMNR Poisoning (Responder)
                           │
                           ▼
             Capture NTLMv2 Authentication Hash
                           │
                           ▼
      Offline Password Cracking (John the Ripper)
                           │
                           ▼
      Compromised User Account (tstark)
                           │
                           ▼
 Kerberoasting (GetUserSPNs + John the Ripper)
                           │
                           ▼
 Compromised Service Account (sqlservice)
                           │
                           ▼
      Privilege Escalation (Domain Admin)
                           │
                           ▼
        DCSync (Impacket secretsdump.py)
                           │
                           ▼
     Extract Domain Administrator Hashes
                           │
                           ▼
      Pass-the-Hash (Impacket psexec.py)
                           │
                           ▼
 NT AUTHORITY\SYSTEM Shell on Domain Controller
                           │
                           ▼
 BloodHound Enumeration & Domain Analysis
                           │
                           ▼
      Complete Active Directory Compromise
```



## Tools & Technologies Used

| Tool | Purpose | Version |
|------|---------|---------|
| Responder | LLMNR/NBT-NS poisoning and NTLMv2 hash capture | 3.2.2.0 |
| John the Ripper | Offline password cracking | Latest |
| Impacket GetUserSPNs | Kerberoasting and SPN enumeration | 0.14.0 |
| Impacket secretsdump | DCSync credential extraction | 0.14.0 |
| Impacket psexec | Pass-the-Hash remote command execution | 0.14.0 |
| BloodHound | Active Directory visualization | 9.2.2 |
| bloodhound-python | Active Directory data collection | Latest |
| Neo4j | Graph database for BloodHound | Latest |
| VMware Workstation Pro | Virtualization platform | 26H1 |
| Kali Linux | Attacker operating system | Rolling |
| Windows Server 2022 | Domain Controller | Lab Environment |
| Windows 10 | Domain Workstation (WS01) | Lab Environment |



## Attack Timeline

| Phase | Attack | Tool | Target | Result | Approx. Time |
|------|--------|------|--------|--------|-------------|
| 1 | LLMNR Poisoning | Responder | WS01 | Captured NTLMv2 hash | ~2 min |
| 2 | Offline Password Cracking | John the Ripper | NTLMv2 Hash | Recovered `tstark` password | ~1 min |
| 3 | Kerberoasting | GetUserSPNs | DC01 | Requested `sqlservice` TGS ticket | ~1 min |
| 4 | Kerberos Ticket Cracking | John the Ripper | TGS Ticket | Recovered `sqlservice` password | ~2 min |
| 5 | Privilege Escalation | Active Directory | Domain | Added `tstark` to Domain Admins | ~1 min |
| 6 | DCSync | secretsdump.py | DC01 | Extracted all domain credentials | ~3 min |
| 7 | Pass-the-Hash | psexec.py | DC01 | Obtained SYSTEM shell | ~1 min |
| 8 | Domain Enumeration | BloodHound | Active Directory | Visualized attack paths | ~2 min |



## Attack Summary

### Initial Access

The attack began by exploiting the Windows Link-Local Multicast Name Resolution (LLMNR) protocol. By poisoning LLMNR responses with Responder, an NTLMv2 authentication hash belonging to the domain user **tstark** was successfully captured.

### Credential Access

The captured NTLMv2 hash was cracked offline using John the Ripper, revealing the plaintext password for the `tstark` account. Using these valid domain credentials, Kerberoasting was performed against the Active Directory environment, resulting in the recovery of the `sqlservice` account password.

### Privilege Escalation

After obtaining privileged credentials during the lab, the `tstark` account was added to the **Domain Admins** group, granting replication privileges within Active Directory.

### Credential Dumping

Using Domain Administrator privileges, a DCSync attack was performed with `secretsdump.py`. This allowed replication of Active Directory credentials directly from the Domain Controller without logging into the server interactively.

### Lateral Movement

The extracted Administrator NTLM hash was used in a Pass-the-Hash attack with `psexec.py`. This resulted in an interactive **NT AUTHORITY\SYSTEM** shell on the Domain Controller.

### Active Directory Enumeration

Finally, `bloodhound-python` collected Active Directory information, which was imported into BloodHound. The resulting graph visualized users, groups, computers, permissions, and privilege escalation paths throughout the MARVEL domain.


## MITRE ATT&CK Mapping

| Attack Technique | MITRE ATT&CK ID |
|-----------------|-----------------|
| LLMNR / NBT-NS Poisoning (Adversary-in-the-Middle) | T1557 |
| Password Cracking | T1110.002 |
| Kerberoasting | T1558.003 |
| Account Discovery | T1087 |
| Permission Group Discovery | T1069 |
| DCSync | T1003.006 |
| Pass-the-Hash | T1550.002 |
| Remote Service Execution | T1021 |



## Lessons Learned & Security Recommendations

The penetration test demonstrated how several common Active Directory misconfigurations can be chained together to achieve complete domain compromise. Although each weakness may appear minor individually, combining them allowed an attacker to progress from an unprivileged position to full Domain Controller compromise within a short period.

The following findings summarize the vulnerabilities identified during the assessment along with their associated risks and recommended mitigations.



## Critical Vulnerabilities Identified

### Finding 1: LLMNR / NBT-NS Poisoning

**Risk Rating:** High

#### Description

The environment allowed Link-Local Multicast Name Resolution (LLMNR) requests to be broadcast across the network. Because these broadcasts were not authenticated, the attacker successfully poisoned the responses using Responder and captured NTLMv2 authentication hashes from domain users.

#### Impact

An attacker connected to the internal network can capture authentication attempts without exploiting software vulnerabilities.

Captured hashes may later be cracked offline or used in relay attacks.

#### Recommendations

- Disable LLMNR through Group Policy.
- Disable NBT-NS where possible.
- Enable SMB Signing.
- Prefer DNS-only name resolution.
- Monitor for LLMNR and NBT-NS poisoning attempts.



### Finding 2: Weak Password Policy

**Risk Rating:** Critical

#### Description

Several domain accounts used weak and predictable passwords that were successfully recovered using dictionary attacks.

Examples recovered during testing included:

```
tstark
Password1!

sqlservice
MYpassword123#
```

#### Impact

Weak passwords allowed attackers to quickly obtain valid domain credentials after capturing authentication hashes.

### Recommendations

- Enforce password complexity requirements.
- Require a minimum password length of at least 14 characters.
- Prevent password reuse.
- Regularly rotate privileged account passwords.
- Deploy Multi-Factor Authentication (MFA) wherever supported.



### Finding 3: Service Account Security (Kerberoasting)

**Risk Rating:** High

#### Description

The SQL service account contained a registered Service Principal Name (SPN) and used a password vulnerable to offline cracking.

Because any authenticated domain user can request Kerberos service tickets, attackers were able to obtain a Ticket Granting Service (TGS) ticket and recover the service account password.

#### Impact

Compromised service accounts may provide attackers with privileged access to servers and critical infrastructure.

#### Recommendations

- Replace traditional service accounts with Group Managed Service Accounts (gMSAs).
- Use long, randomly generated passwords for service accounts.
- Monitor for unusual Kerberos TGS requests.
- Regularly audit registered SPNs.



### Finding 4: Excessive Privileges

**Risk Rating:** Critical

#### Description

Once elevated to the Domain Admins group, the attacker obtained unrestricted administrative privileges across the domain.

These privileges enabled credential replication and complete Active Directory compromise.

#### Impact

Compromise of privileged accounts allows attackers to fully control the Active Directory environment.

#### Recommendations

- Apply the Principle of Least Privilege.
- Minimize Domain Administrator membership.
- Implement Just-In-Time (JIT) administration.
- Use Privileged Access Management (PAM).
- Continuously audit privileged group memberships.



### Finding 5: DCSync Credential Replication

**Risk Rating:** Critical

#### Description

The attacker successfully executed a DCSync attack using Domain Administrator privileges.

Instead of accessing the Active Directory database directly, the Domain Controller replicated password hashes through its normal replication mechanism.

#### Impact

The attacker obtained:

- Administrator NTLM hashes
- User NTLM hashes
- Kerberos keys
- Password history (where available)

This effectively resulted in complete compromise of Active Directory credentials.

### Recommendations

- Restrict replication permissions.
- Continuously monitor Directory Replication Service (DRS) requests.
- Alert on unauthorized DCSync activity.
- Regularly audit accounts with replication privileges.



### Finding 6: Pass-the-Hash Authentication

**Risk Rating:** High

#### Description

The Administrator NTLM hash extracted through DCSync was successfully used to authenticate to the Domain Controller without requiring the plaintext password.

#### Impact

Possession of an NTLM hash can provide the same level of access as knowing the user's password.

#### Recommendations

- Enable Credential Guard.
- Restrict NTLM authentication where possible.
- Use Windows Defender Remote Credential Guard.
- Regularly rotate privileged account passwords.
- Monitor NTLM authentication events.



## Additional Security Controls

### Detection & Monitoring

Organizations should continuously monitor Active Directory for suspicious activity.

Recommended detections include:

- LLMNR and NBT-NS poisoning attempts
- Excessive Kerberos TGS requests
- DCSync replication activity
- Pass-the-Hash authentication
- Privileged group membership changes
- Creation of new administrative accounts
- Remote service creation through PsExec-like tools



### Active Directory Hardening

To reduce the attack surface, the following hardening measures should be implemented:

- Disable LLMNR and NBT-NS.
- Enable SMB Signing.
- Enforce LDAP Signing.
- Require strong password policies.
- Deploy Multi-Factor Authentication.
- Use Group Managed Service Accounts (gMSAs).
- Remove inactive user accounts.
- Regularly audit privileged accounts.
- Restrict local administrator rights.
- Apply Microsoft's Security Baselines.



### Incident Response Recommendations

If an organization suspects compromise of an Active Directory environment similar to this attack, the following actions should be taken immediately:

1. Isolate compromised systems from the network.
2. Reset passwords for all compromised accounts.
3. Reset the **KRBTGT** account password twice.
4. Remove unauthorized privileged accounts.
5. Review Domain Admin group membership.
6. Audit replication permissions.
7. Review authentication logs for lateral movement.
8. Investigate SMB, Kerberos, and LDAP activity.
9. Rebuild compromised systems if necessary.
10. Perform a complete Active Directory security assessment.




## Conclusion

This Active Directory penetration testing lab successfully demonstrated a complete attack chain beginning with an initial foothold and ending in full Domain Controller compromise.

The assessment showed how multiple common Active Directory weaknesses can be combined to compromise an enterprise environment. The attack started with LLMNR poisoning to capture an NTLMv2 authentication hash, followed by offline password cracking to obtain valid domain credentials. These credentials enabled a Kerberoasting attack against a vulnerable service account, allowing recovery of an additional privileged password.

After obtaining administrative privileges, a DCSync attack was performed to replicate Active Directory credentials directly from the Domain Controller. The extracted Administrator NTLM hash was then used in a Pass-the-Hash attack to obtain an interactive **NT AUTHORITY\SYSTEM** shell on the Domain Controller. Finally, BloodHound was used to enumerate and visualize the Active Directory environment, confirming privilege relationships and attack paths.

The complete attack chain demonstrated how a single compromised user account can rapidly escalate into complete domain compromise when multiple security weaknesses exist simultaneously.

This assessment highlights several important security lessons:

- Legacy protocols such as LLMNR significantly increase the attack surface.
- Weak passwords remain one of the most common causes of credential compromise.
- Service accounts require strong passwords and regular security reviews.
- Privileged accounts should be tightly controlled and continuously monitored.
- Active Directory replication permissions must be restricted to authorized accounts only.
- Continuous monitoring and defense-in-depth are essential to detect and prevent credential-based attacks.

Although this assessment was performed within a controlled laboratory environment, the techniques demonstrated closely resemble those used by real-world attackers during Active Directory intrusions. Proper system hardening, strong authentication policies, continuous monitoring, and adherence to security best practices are essential to reducing the likelihood and impact of such attacks in production environments.


## Skills Demonstrated

Throughout this assessment, the following cybersecurity skills were successfully demonstrated:

#### Active Directory Enumeration

- Domain reconnaissance
- User and group enumeration
- Service Principal Name (SPN) enumeration
- Active Directory relationship analysis

#### Credential Attacks

- LLMNR Poisoning
- NTLMv2 Hash Capture
- Offline Password Cracking
- Kerberoasting
- DCSync
- Pass-the-Hash

#### Privilege Escalation

- Service account compromise
- Domain Administrator privilege escalation
- SYSTEM-level remote code execution

#### Active Directory Analysis

- BloodHound data collection
- Attack path visualization
- Privilege relationship mapping
- Graph-based security analysis

#### Defensive Knowledge

- Active Directory hardening
- Credential protection
- Password policy recommendations
- Detection and monitoring strategies
- Incident response considerations



## References

- [Microsoft – Active Directory Security Best Practices](https://learn.microsoft.com/)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [Fortra Impacket Documentation](https://github.com/fortra/impacket)
- [BloodHound Documentation](https://bloodhound.specterops.io/)
- [Neo4j Documentation](https://neo4j.com/docs/)
- [John the Ripper Documentation](https://www.openwall.com/john/)
- [Responder GitHub Repository](https://github.com/lgandx/Responder)



### Appendix A – MITRE ATT&CK Mapping

| Technique | MITRE ATT&CK ID |
|-----------|-----------------|
| LLMNR / NBT-NS Poisoning | T1557 |
| Password Cracking | T1110.002 |
| Kerberoasting | T1558.003 |
| Active Directory Account Discovery | T1087 |
| Permission Group Discovery | T1069 |
| DCSync | T1003.006 |
| Pass-the-Hash | T1550.002 |
| Remote Service Execution | T1021 |



### Appendix B – Lab Environment

| Component | Configuration |
|----------|---------------|
| Hypervisor | VMware Workstation Pro 26H1 |
| Attacker Machine | Kali Linux (Rolling Release) |
| Domain Controller | Windows Server 2022 |
| Client Machine | Windows 11 Enterprise |
| Domain Name | MARVEL.local |
| Domain Controller | DC01 |
| Workstation | WS01 |



### Appendix C – Report Information

**Report Title:** Active Directory Penetration Testing Lab Report

**Author:** Sumanth A (CobraSummy)

**Lab Environment:** Private Virtual Lab - VMware Workstation Pro 

**Assessment Type:** Authorized Penetration Testing (Educational)

**Report Date:** June 2026

**Total Attack Time:** Approximately 15 Minutes

**Report Version:** 1.0
