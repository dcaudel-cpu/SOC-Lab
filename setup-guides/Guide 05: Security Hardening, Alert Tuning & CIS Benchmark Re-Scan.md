# Guide 05: Incident Response, Security Hardening & CIS Benchmark Re-Scan

## Overview
Following the adversary simulations in **Guide 04**, this phase transitions from detection to **containment, remediation, and proactive system hardening**. By addressing the attack vectors exploited during offensive testing, tuning noisy SIEM rules (such as the `svchost.exe` false positives on `Win-DC01`), and applying Group Policy security baselines, we significantly reduce the attack surface of `Company.local` and measure our progress against the initial **25% CIS Benchmark baseline**.

---

## Phase 1: Post-Simulation Containment & Eradication
Before applying preventative controls, remove all artifacts left behind during the Guide 04 offensive testing phase on **Win-DC01** (`192.168.56.10`).

### 1. Remove Malicious Registry Persistence
Open an elevated PowerShell prompt on `Win-DC01` and remove the Run-key persistence created during testing:
```powershell
# Inspect current Run keys
Get-ItemProperty -Path "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run"
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run"

# Remove the test payload persistence entry (replace <PayloadName> with your specific key)
Remove-ItemProperty -Path "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "<PayloadName>" -ErrorAction SilentlyContinue
Remove-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "<PayloadName>" -ErrorAction SilentlyContinue
```

### 2. Purge Payload Binaries & Temporary Scripts
Delete any `msfvenom` executables or staging scripts dropped on disk:
```powershell
# Example cleanup of staging directories
Remove-Item -Path "C:\Temp\*.exe", "C:\Users\Public\*.exe", "C:\Users\Public\*.ps1" -Force -ErrorAction SilentlyContinue
```

### 3. Verify FIM & Wazuh Alerting
* Navigate to **Wazuh Dashboard > Security Events / Integrity Monitoring**.
* Confirm that `syscheck` logged the deletion of the malicious binaries and modification of the `Run` registry keys.

---

## Phase 2: Wazuh Alert Tuning (Reducing False Positives)
During baseline operations and testing on a Windows Server 2025 Domain Controller, legitimate administrative and Active Directory background tasks frequently trigger high-severity rules—such as **Rule 61618** (`Sysmon - Suspicious Process - svchost.exe`).

### Tuning Rule 61618 in `local_rules.xml`
Rather than disabling Rule 61618 globally, create a granular exception on your **Wazuh Manager** that suppresses alerts *only* when `svchost.exe` is legitimately spawned from `System32` by `services.exe` with a valid `-k` service group.

1. SSH into your **Wazuh Manager** and edit `/var/ossec/etc/rules/local_rules.xml`:
```xml
<group name="sysmon,windows,tuning,">

  <!-- Suppress benign svchost.exe Rule 61618 alerts on Win-DC01 -->
  <rule id="100105" level="0">
    <if_sid>61618</if_sid>
    <hostname>Win-DC01</hostname>
    <field name="win.eventdata.image" type="pcre2">(?i)^C:\\Windows\\System32\\svchost\.exe\$</field>
    <field name="win.eventdata.parentImage" type="pcre2">(?i)^C:\\Windows\\System32\\services\.exe\$</field>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)-k\s+</field>
    <description>Tuning: Suppress legitimate svchost.exe spawned by services.exe on Win-DC01.</description>
  </rule>

</group>
```

2. Validate the XML syntax and restart the Wazuh Manager:
```bash
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
```

<img alt="Local Rules Tuning" src="https://githubusercontent.com" />

---

## Phase 3: Active Directory & Endpoint Hardening (`Company.local`)
To remediate the vulnerabilities exposed by `netexec`, `smbclient`, and encoded PowerShell execution in Guide 04, apply targeted hardening via **Group Policy Management (`gpmc.msc`)** on `Win-DC01`.

### 1. Enforce SMB Signing & Disable SMBv1
Prevents SMB relay attacks and unauthenticated enumeration.
* **GPO Path:** `Default Domain Policy` (or a dedicated `Server Hardening GPO`) -> `Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options`
* **Configure:**
  * `Microsoft network server: Digitally sign communications (always)` -> **Enabled**
  * `Microsoft network client: Digitally sign communications (always)` -> **Enabled**
* Disable legacy SMBv1 via PowerShell:
```powershell
Set-SmbServerConfiguration -EnableSMB1Protocol \$false -Force
```

### 2. Harden Account Lockout & Password Policies
Mitigates brute-force and password spraying attacks (`netexec smb`).
* **GPO Path:** `Computer Configuration > Policies > Windows Settings > Security Settings > Account Policies`
* **Password Policy:**
  * Minimum password length: **14 characters**
  * Password complexity: **Enabled**
* **Account Lockout Policy:**
  * Account lockout threshold: **5 invalid logon attempts**
  * Account lockout duration: **15 minutes**
  * Reset account lockout counter after: **15 minutes**

<img alt="Password Policy Hardening" src="https://githubusercontent.com" />

### 3. Restrict Anonymous Enumeration
Blocks null-session enumeration of domain users and shares.
* **GPO Path:** `Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options`
* **Configure:**
  * `Network access: Do not allow anonymous enumeration of SAM accounts` -> **Enabled**
  * `Network access: Do not allow anonymous enumeration of SAM accounts and shares` -> **Enabled**

<img alt="Active Directory Security Options Hardening" src="https://githubusercontent.com" />

### 4. PowerShell Constrained Language Mode & Script Block Logging
Neutralizes obfuscated/encoded PowerShell payloads and ensures full command visibility in Wazuh.
* **GPO Path:** `Computer Configuration > Policies > Administrative Templates > Windows Components > Windows PowerShell`
* **Configure:**
  * `Turn on PowerShell Script Block Logging` -> **Enabled**
  * `Turn on Module Logging` -> **Enabled** (Module Names: `*`)

### 5. Enforce Group Policy Updates
Apply all new GPO configurations immediately on `Win-DC01`:
```powershell
gpupdate /force
```

---

## Phase 4: Re-Validation & CIS Benchmark Re-Scan

### 1. Re-Run Offensive Validation (Purple Team Check)
From **Kali Linux**, re-run the enumeration and brute-force commands from Guide 04 to verify the controls work:
```bash
# Verify SMB signing is now required and anonymous null sessions fail
netexec smb 192.168.56.10 -u '' -p '' --shares
```
* **Expected Result:** Anonymous share enumeration is denied (`STATUS_ACCESS_DENIED`), and `signing:True` is enforced.

<img alt="Re-Test Anonymous SMB & SAMR Enumeration" src="https://githubusercontent.com" />

<img alt="Account Lockout Validation on Kali" src="https://githubusercontent.com" />

<img alt="Wazuh Account Lockout Alert" src="https://githubusercontent.com" />

### 2. Execute CIS Benchmark Re-Scan (Wazuh SCA)
1. Restart the Wazuh Agent on `Win-DC01` to trigger an immediate Security Configuration Assessment (SCA) scan:
```powershell
Restart-Service -Name WazuhSvc
```

---

## Summary of Remediation Impact

### CIS Benchmark Re-Scan Results (25% -> 27%)
After enforcing the targeted Group Policy hardening controls and restarting the Wazuh Agent on `Win-DC01`, the Security Configuration Assessment (SCA) re-scan confirmed an increase in compliance from **25% to 27%**.

<img alt="CIS Benchmark SCA Increase to 27%" src="https://githubusercontent.com" />

#### Why Risk-Based Remediation Was Prioritized Over Score Chasing
* **Targeted Attack-Path Closure:** Rather than blindly applying hundreds of restrictive CIS member-server checks that risk breaking Active Directory authentication, DNS, or replication, remediation focused strictly on closing the high-severity attack vectors exploited in **Guide 04** (`netexec` null-session enumeration, SMB relay, credential brute-forcing, and unlogged PowerShell execution).
* **Operational Availability:** Because CIS Windows Server policies contain over 300 granular checks, resolving critical SMB, authentication, and logging vulnerabilities yields a measured percentage increase while maintaining 100% Domain Controller stability.




