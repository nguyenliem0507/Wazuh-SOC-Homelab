# 03 — Detection & Custom Rules

## Overview

This document describes how Wazuh detected the simulated attacks and the custom detection rule written to identify ransomware behavior.

---

## 1. Brute-Force Detection

### Wazuh Built-in Detection

After the SMB brute-force attack, Wazuh automatically generated alerts based on:
- **Event ID 4625** — Windows Failed Logon
- **Event ID 4624** — Windows Successful Logon

The Wazuh Dashboard showed a clear spike in security events at the exact time of the attack:

![Security Events Dashboard](../images/detections/wazuh_security_events.jpg)

**Key metrics recorded:**
- Total events: **525**
- Authentication failures: **3**
- Authentication successes: **2**

---

## 2. Pass-the-Hash Detection (NTLM)

### Detection Details

Wazuh flagged the successful SMB authentication via NTLM as a potential **Pass-the-Hash** attack.

![NTLM Detection](../images/detections/ntlm_detection.jpg)

**Alert:** `Successful Remote Logon Detected - User:\Administrator - NTLM authentication, possible pass-the-hash attack.`
- **Rule ID:** `92652`
- **Rule Level:** `6`

### MITRE ATT&CK Automatic Mapping

Wazuh automatically correlated this event with the MITRE ATT&CK framework:

![MITRE ATT&CK Mapping](../images/detections/mitre_attack_mapping.jpg)

| Field | Value |
|-------|-------|
| `rule.mitre.id` | **T1550.002**, T1078.002 |
| `rule.mitre.tactic` | Defense Evasion, Lateral Movement, Persistence, Privilege Escalation, Initial Access |
| `rule.mitre.technique` | **Pass the Hash**, Domain Accounts |
| `rule.pci_dss` | 10.2.5 |
| `rule.nist_800_53` | AC.7, AU.14 |
| `rule.gdpr` | IV_32.2 |
| `rule.hipaa` | 164.312.b |

---

## 3. Custom Detection Rule — Ransomware (Shadow Copy Deletion)

### Motivation

While Wazuh has built-in rules for many attack techniques, `vssadmin delete shadows` is a highly specific ransomware indicator that required a custom rule to produce a **Critical (Level 15)** alert.

### Rule Implementation

The rule was written in `/var/ossec/etc/rules/local_rules.xml` on the Wazuh Server:

```xml
<group name="ransomware,windows,">

  <rule id="100001" level="15">
    <if_group>windows</if_group>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)vssadmin.*delete.*shadows</field>
    <description>[Homelab] RANSOMWARE ACTIVITY: Shadow Copy deletion detected</description>
    <mitre>
      <id>T1490</id>
    </mitre>
  </rule>

</group>
```

**Rule Logic Explained:**
| Element | Purpose |
|---------|---------|
| `id="100001"` | Custom rule ID (must be > 100000 for local rules) |
| `level="15"` | Critical severity — highest level in Wazuh |
| `if_group>windows` | Only applies to Windows endpoint events |
| `pcre2` regex | Case-insensitive match on `vssadmin.*delete.*shadows` |
| `T1490` | MITRE ATT&CK — Inhibit System Recovery |

![Custom Rule Code](../images/detections/custom_rule_code.jpg)

### Applying the Rule

```bash
# Restart Wazuh Manager to load the new rule
sudo systemctl restart wazuh-manager
```

### Result

After re-executing `vssadmin.exe delete shadows /all /quiet` on the Windows 10 machine, Wazuh triggered the custom alert at **Level 15 (Critical)**:

![Custom Alert Triggered](../images/detections/custom_alert_triggered.jpg)

---

## 4. Incident Response Playbook (Basic)

When the `[Homelab] RANSOMWARE ACTIVITY` alert is triggered, a SOC Analyst should:

1. **Triage:** Verify the alert source — confirm the process was `vssadmin.exe` initiated by an unexpected user or process.
2. **Contain:** Isolate the affected endpoint from the network immediately.
3. **Investigate:** Check for other ransomware indicators:
   - Unusual file encryption activity (mass file modification)
   - Outbound C2 connections (suspicious network traffic)
   - Privilege escalation events preceding the alert
4. **Escalate:** If confirmed ransomware, escalate to Tier 2/3 and activate the Incident Response plan.
5. **Document:** Log all findings in the ticketing system (e.g., Jira, TheHive).

---

## Summary

| Attack | MITRE ID | Wazuh Rule | Level | Detected |
|--------|----------|------------|-------|---------|
| SMB Brute-Force | T1110 | Built-in | 5 | ✅ |
| Pass-the-Hash (NTLM) | T1550.002 | 92652 | 6 | ✅ |
| Shadow Copy Deletion | T1490 | **100001 (Custom)** | **15** | ✅ |
