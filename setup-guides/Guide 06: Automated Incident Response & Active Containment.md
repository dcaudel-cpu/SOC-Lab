# Guide 06: Automated Incident Response & Active Containment

## 🎯 Objective
Transition the SOC homelab from passive detection to **automated incident containment** using **Wazuh Active Response**. By configuring automated firewall blocking on `Win-DC01`, the SIEM will immediately isolate malicious source IPs upon detecting high-confidence attack patterns—reducing Mean Time to Respond (MTTR) from minutes to seconds.

---

## ⚠️ Pre-Lab Safety Check: VirtualBox Snapshots
Before modifying firewall or Wazuh configuration files:
1. Shut down or pause both virtual machines (`Kali Linux` and `Win-DC01`).
2. Take a clean VirtualBox snapshot on both VMs labeled **`Post-Guide05-Hardened`**.
3. Boot both VMs and verify the Wazuh Agent on `Win-DC01` shows **Active** in the Wazuh Dashboard.

---

## 🛠️ Phase 1: Enable Windows Firewall & Active Response Prerequisites

Wazuh's built-in Windows firewall script (`netsh.exe`) relies on the Windows Defender Firewall service running on the monitored endpoint.

### On `Win-DC01` (PowerShell as Administrator):
1. Verify that Windows Defender Firewall is enabled across profiles (or at least the Domain/Private profile):
   ```powershell
   Get-NetFirewallProfile | Select-Object Name, Enabled
   ```
2. If disabled, enable the Domain profile (ensure Wazuh Agent ports `1514/tcp` and `1515/tcp` are not blocked):
   ```powershell
   Set-NetFirewallProfile -Profile Domain,Private -Enabled True
   ```
3. Confirm the Wazuh Active Response scripts exist on the Windows agent:
   ```powershell
   Get-ChildItem "C:\Program Files (x86)\ossec-agent\active-response\bin\"
   ```
   *(Verify that `netsh.exe` or `route-null.exe` is present in this directory.)*

---

## ⚙️ Phase 2: Configure Wazuh Active Response on the Manager

Configure the Wazuh Manager to trigger the `firewall-drop` command on `Win-DC01` whenever a high-severity authentication or brute-force rule fires (such as multiple failed RDP/SMB logon attempts, MITRE `T1110`).

### On the Wazuh Server (Kali Linux):
1. Open `/var/ossec/etc/ossec.conf` for editing:
   ```bash
   sudo nano /var/ossec/etc/ossec.conf
   ```
2. Locate the `<active-response>` section (or add the following block inside `<ossec_config>`):
   ```xml
   <active-response>
     <disabled>no</disabled>
     <command>firewall-drop</command>
     <location>local</location>
     <rules_id>60122, 60204</rules_id>
     <timeout>300</timeout>
   </active-response>
   ```
   * **`command` (`firewall-drop`):** Executes the native Windows firewall block script on the target agent.
   * **`location` (`local`):** Applies the block directly on the endpoint (`Win-DC01`) that generated the alert.
   * **`rules_id` (`60122, 60204`):** Triggers on Windows logon failure (`60122`) and multiple failed logon / brute-force detection (`60204`). *(Tip: Start with `60204` if you only want it to trigger after repeated failures rather than a single typo.)*
   * **`timeout` (`300`):** Automatically removes the firewall block after **300 seconds (5 minutes)** to prevent permanent self-lockouts during testing.

3. Restart the Wazuh Manager to apply the configuration:
   ```bash
   sudo systemctl restart wazuh-manager
   ```

---

## ⚔️ Phase 3: Simulate Brute-Force Attack & Validate Containment

Simulate an SMB/Windows authentication brute-force attack from Kali Linux against `Win-DC01` to verify end-to-end automated containment.

### 1. Run Brute-Force Simulation from Kali Linux:
```bash
netexec smb 192.168.56.X -u Administrator -p /usr/share/wordlists/fasttrack.txt --continue-on-success
```
*(Replace `192.168.56.X` with the static IP address of `Win-DC01`.)*

### 2. Observe Automated Containment:
* **On Kali Linux:** Notice that `netexec` initially logs `STATUS_LOGON_FAILURE` attempts, and then abruptly stalls or throws connection timeout errors once the threshold is crossed.
* **Verify Block from Kali:** Attempt a quick `ping` or `nmap` scan against `Win-DC01`—traffic from Kali's IP (`192.168.56.Y`) is now actively dropped by `Win-DC01`.

### 3. Verify Windows Firewall Rule on `Win-DC01`:
Open PowerShell on `Win-DC01` and inspect the dynamically created Wazuh firewall rule:
```powershell
Get-NetFirewallRule -DisplayName "*WAZUH*" | Select-Object DisplayName, Enabled, Action, Direction
```
*(Capture a screenshot of this output showing `Action : Block` for your lab documentation.)*

### 4. Inspect Active Response Logs in Wazuh:
* Check the local Active Response log on `Win-DC01`:
  ```powershell
  Get-Content "C:\Program Files (x86)\ossec-agent\active-response\active-responses.log" -Tail 10
  ```
* In the **Wazuh Dashboard**, navigate to **Threat Hunting / Events** and filter for `rule.groups: "active_response"` to confirm the alert and containment execution timestamps.

---

## 🧹 Phase 4: Verification of Auto-Expiration & Cleanup

1. Wait **5 minutes (300 seconds)** for the `<timeout>` value to expire.
2. Re-run the firewall check on `Win-DC01` to confirm Wazuh cleanly removed the temporary block:
   ```powershell
   Get-NetFirewallRule -DisplayName "*WAZUH*"
   ```
3. Verify connectivity from Kali Linux returns to normal (`ping -c 2 192.168.56.X`).

---

## 📌 Key SOC Takeaways
* **Reduced Dwell Time:** Automated containment stops brute-force and lateral movement attempts in real time without waiting for manual analyst intervention.
* **Safe Automation Guardrails:** Using a timed block (`<timeout>300</timeout>`) and targeting high-confidence correlation rules (`60204`) balances rapid defense against accidental denial-of-service on legitimate administrative traffic.
