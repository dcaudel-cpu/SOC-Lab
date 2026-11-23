# Guide 03: Sysmon Deployment & File Integrity Monitoring (FIM) 

This guide documents tuning noisy Windows background processes out of custom Sysmon rules in **Wazuh**, configuring real-time **File Integrity Monitoring (FIM)** on `Win-DC01`, and reviewing **Security Configuration Assessment (SCA)** baseline checks to harden the domain controller.

---

## 🎯 Objectives
* Suppress benign Windows background executables from triggering false positives under base Sysmon Rule `100100` in `local_rules.xml`.
* Enable real-time File Integrity Monitoring (`syscheck`) on sensitive system folders and Windows startup registry keys.
* Review CIS Benchmark findings in the Wazuh SCA module and establish a documented security baseline for `Win-DC01`.

---

## 🛠️ Step 1: Tune Noisy Processes in `local_rules.xml`

Once Sysmon Event ID 1 (Process Creation) is routed into Wazuh, routine Windows services generate high alert volume. Adding child suppression rules with `<level>0</level>` silences known benign activity while preserving alerts for anomalous execution.

1. **Open `local_rules.xml` on the Wazuh Manager (`kali`):**
   ```bash
   sudo nano /var/ossec/etc/rules/local_rules.xml
   ```
2. **Add child suppression rules referencing `100100`:**
   ```xml
   <group name="sysmon,tuning,">
     <!-- Suppress routine Windows background executables -->
     <rule id="100101" level="0">
       <if_sid>100100</if_sid>
       <field name="win.eventdata.image" type="pcre2">(?i)C:\\Windows\\System32\\(svchost|taskhostw|conhost|backgroundTaskHost)\.exe\$</field>
       <description>Sysmon - Suppress routine Windows background processes</description>
     </rule>

     <!-- Suppress Windows Defender signature updates -->
     <rule id="100102" level="0">
       <if_sid>100100</if_sid>
       <field name="win.eventdata.image" type="pcre2">(?i)C:\\ProgramData\\Microsoft\\Windows Defender\\Platform\\.*\\MpCmdRun\.exe\$</field>
       <description>Sysmon - Suppress Windows Defender update utility</description>
     </rule>
   </group>
   ```
3. **Validate rule syntax and restart `wazuh-manager`:**
   ```bash
   sudo /var/ossec/bin/wazuh-analysisd -t
   sudo systemctl restart wazuh-manager
   ```

---

## 📂 Step 2: Configure Real-Time File Integrity Monitoring (FIM)

Wazuh's `syscheck` module detects unauthorized file drops, hosts file tampering, and registry persistence mechanisms on `Win-DC01`.

1. **Edit the agent configuration on `Win-DC01`:** Open `C:\Program Files (x86)\ossec-agent\ossec.conf` in an elevated editor.
2. **Configure real-time directory and registry monitoring inside `<syscheck>`:**
   ```xml
   <syscheck>
     <disabled>no</disabled>
     <frequency>43200</frequency>
     <scan_on_start>yes</scan_on_start>

     <!-- Monitor critical system directories in real time -->
     <directories check_all="yes" realtime="yes" report_changes="yes">C:\Windows\System32\drivers\etc</directories>
     <directories check_all="yes" realtime="yes">C:\Windows\System32\WindowsPowerShell\v1.0</directories>
     <directories check_all="yes" realtime="yes">C:\Users\Administrator\Desktop</directories>

     <!-- Monitor Windows Run/RunOnce persistence keys -->
     <windows_registry>HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run</windows_registry>
     <windows_registry>HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunOnce</windows_registry>
   </syscheck>
   ```
3. **Restart the Wazuh Agent service on `Win-DC01`:**
   ```powershell
   Restart-Service -Name WazuhSvc
   ```

---

## 📋 Step 3: Review Security Configuration Assessment (SCA) Baselines

1. **Navigate to SCA in the Wazuh Dashboard:** Go to **Modules > Security Configuration Assessment** and select the `Win-DC01` agent.
2. **Inspect the CIS Benchmark policy:** Review failed checks under the **CIS Microsoft Windows Server Benchmark** to identify high-priority misconfigurations (e.g., enabled `Guest` account, SMBv1 status, or weak password lockout thresholds).
3. **Verify telemetry:** Create a test file on `C:\Users\Administrator\Desktop\fim-test.txt` and confirm a Rule `554` (File added to the system) alert appears under **Modules > Integrity Monitoring**.

---

## 🔧 Troubleshooting & Lessons Learned

* **Manager Fails to Restart After Rule Edits:** Always run `sudo /var/ossec/bin/wazuh-analysisd -t` before restarting `wazuh-manager`. Unescaped backslashes in PCRE2 regex patterns (`\` instead of `\\`) are the most common cause of XML validation errors.
* **FIM Alerts Delayed:** Standard periodic scans run every 12 hours (`43200` seconds). Ensure `realtime="yes"` is explicitly set on directories where immediate alerting is required.

---

## 📸 Verification Screenshots
<img width="3403" height="826" alt="03-FIM Test Screenshot" src="https://github.com/user-attachments/assets/64b81458-435b-404e-96d7-c4a8ef552c77" />
<img width="1707" height="1153" alt="03-sca-baseline-25-percent" src="https://github.com/user-attachments/assets/c88eb4a7-a8a8-4fef-a6db-4f82ff3e00f7" />


