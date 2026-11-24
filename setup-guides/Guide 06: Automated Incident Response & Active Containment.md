# Guide 06: Automated Threat Containment with Wazuh Active Response

## Overview
In this phase of the homelab, we transition from passive detection to **automated incident response (SOAR-style containment)**. Using **Wazuh Active Response**, the Wazuh Manager automatically detects SMB brute-force attempts targeting the Windows Domain Controller (`Win-DC01`) and instructs the Windows agent to dynamically block the attacker's IP address via Windows Defender Firewall (`netsh.exe`).

### Lab Architecture & Targets
* **Wazuh Manager & Attacker Host:** Kali Linux
* **Target Endpoint:** `Win-DC01` (`DC-COMPANY`, `192.168.56.10`)
* **Trigger Rules:**
  * `60122` — Windows Logon Failure (`Event ID 4625` — Unknown user or bad password)
  * `60204` — Multiple Windows Logon Failures (Brute-force threshold)
* **Containment Action:** Windows Defender Firewall IP Block (`netsh.exe`) with a `300`-second (5-minute) timeout

---

## Phase 1: Configure Wazuh Active Response on the Manager

1. Open `/var/ossec/etc/ossec.conf` on the Wazuh Manager (Kali Linux) and verify that the `netsh` command is defined under the `<command>` section:
   ```xml
   <command>
     <name>netsh</name>
     <executable>netsh.exe</executable>
     <timeout_allowed>yes</timeout_allowed>
   </command>
   ```
   > **Note:** Linux environments typically use `firewall-drop` (`iptables`/`nftables`), which is incompatible with Windows endpoints. Because `<location>local</location>` executes the response directly on the targeted agent (`Win-DC01`), `netsh` must be used instead.

2. Add the `<active-response>` block inside `<ossec_config>` to trigger on rules `60122` and `60204`:
   ```xml
   <active-response>
     <disabled>no</disabled>
     <command>netsh</command>
     <location>local</location>
     <rules_id>60122,60204</rules_id>
     <timeout>300</timeout>
   </active-response>
   ```

3. Validate the configuration syntax and restart the Wazuh Manager:
   ```bash
   sudo /var/ossec/bin/wazuh-analysisd -t
   sudo systemctl restart wazuh-manager
   ```

---

## Phase 2: Simulate an SMB Brute-Force Attack

1. From Kali Linux, launch an SMB authentication attack against `Win-DC01` using `netexec`.
   > **Operational Safety Tip:** When testing against a Domain Controller hardened with CIS Benchmarks, avoid targeting `-u Administrator` directly, as Windows' local Account Lockout Policy will lock the built-in Administrator account independently of the firewall block. Use a non-existent or dedicated test user (`-u fakeuser`) to safely trigger `Event ID 4625` (`Rule 60122`).

   ```bash
   netexec smb 192.168.56.10 -u fakeuser -p 'WrongPassword123!'
   ```

2. **Observe Automated Containment:**
   `netexec` initially resolves the SMB banner for `DC-COMPANY`, then immediately stalls with connection timeouts (`The NETBIOS connection with the remote host timed out`) as Windows Defender Firewall drops all subsequent packets from Kali's IP.

   ![Kali SMB Brute-Force Blocked](./images/06-01-netexec-smb-bruteforce-blocked.png)

---

## Phase 3: Verify Detection & Firewall Containment

1. **Inspect Logon Failure Alerts in Wazuh:**
   Navigate to **Explore > Threat Hunting** in the Wazuh Dashboard and filter for `rule.id:(60122 OR 60204)`. Because blocking the Wazuh Manager's IP temporarily pauses agent log shipping on port `1514`, buffered Windows `Event ID 4625` alerts flush into the dashboard once communication resumes.

   ![Wazuh Logon Failure Alerts](./images/06-02-wazuh-logon-failure-alerts.png)

2. **Verify Windows Defender Firewall Block on `Win-DC01`:**
   Open PowerShell as Administrator on `Win-DC01` and confirm Wazuh dynamically created Inbound and Outbound block rules:
   ```powershell
   Get-NetFirewallRule -DisplayName "*WAZUH*" | Select-Object DisplayName, Enabled, Action, Direction
   ```

   ![Win-DC01 Wazuh Firewall Block](./images/06-03-win-dc01-wazuh-firewall-block.png)

3. **Verify Local Active Response Execution Logs:**
   Inspect `active-responses.log` on `Win-DC01` to confirm `netsh.exe` executed upon receiving the Manager's trigger:
   ```powershell
   Select-String -Path "C:\Program Files (x86)\ossec-agent\active-response\active-responses.log" -Pattern "netsh.exe" | Select-Object -Last 6 | ForEach-Object { \$_.Line -replace '\{.*\}', '' }
   ```

   ![Win-DC01 Active Response Log](./images/06-04-win-dc01-active-response-log.png)

---

## Phase 4: Verification of Cleanup & Restored Connectivity

1. **Firewall Rule Cleanup:**
   After the 300-second `<timeout>` window elapses (or following manual cleanup when multi-threaded probes generate simultaneous duplicate rules), verify that no active `WAZUH` block rules remain on `Win-DC01`:
   ```powershell
   Remove-NetFirewallRule -DisplayName "*WAZUH*"
   Get-NetFirewallRule -DisplayName "*WAZUH*"
   ```

2. **Verify Restored Network Connectivity from Kali:**
   Confirm ICMP/network connectivity between Kali and `Win-DC01` returns to normal:
   ```bash
   ping -c 2 192.168.56.10
   ```

   ![Kali Ping Restored](./images/06-05-kali-ping-restored.png)

3. **Confirm Active Response Audit Events in Wazuh Dashboard:**
   In **Explore > Threat Hunting**, filter by `rule.groups:"active_response"` to verify the full detection-to-containment audit trail in the SIEM.

   ![Wazuh Dashboard Active Response](./images/06-06-wazuh-dashboard-active-response.png)

---

## Key Takeaways & Engineering Lessons
* **OS-Specific Active Response Binaries:** Always match the Active Response `<command>` to the target endpoint OS (`netsh` or `win_route-null` for Windows vs. `firewall-drop` for Linux).
* **Manager/Attacker Co-Location Impact:** Hosting the Wazuh Manager on the same Kali VM used for attack simulations causes the Windows agent to temporarily buffer telemetry while the Manager's IP is blocked at the firewall. In production, the SIEM Manager IP should always be explicitly whitelisted under `<global><white_list>`.
* **Defense-in-Depth Interaction:** Windows CIS Account Lockout policies and Wazuh Active Response operate independently—combining host-level account hardening with automated network containment stops both credential guessing and follow-on lateral movement.

## Verification Screenshots
<img width="1711" height="901" alt="06-06-wazuh-dashboard-active-response" src="https://github.com/user-attachments/assets/4b6342ab-12e8-4315-937c-76e29e58b137" />
<img width="3405" height="627" alt="06-05-kali-ping-restored" src="https://github.com/user-attachments/assets/723e7c9d-04b9-4634-af37-9c0054aad106" />
<img width="1726" height="679" alt="06-04-win-dc01-active-response-log" src="https://github.com/user-attachments/assets/59a541f2-cdc3-47c7-904b-59ea2f018d90" />
<img width="1516" height="615" alt="06-03-win-dc01-wazuh-firewall-block" src="https://github.com/user-attachments/assets/6aebb8f9-223d-45f9-9f90-2215bede2f51" />
<img width="1747" height="873" alt="06-02-wazuh-logon-failure-alerts" src="https://github.com/user-attachments/assets/377e3c25-6b1d-4648-bc33-c99f4bdc8676" />
<img width="1657" height="900" alt="06-01-netexec-smb-bruteforce-blocked png" src="https://github.com/user-attachments/assets/eed63f19-2d60-41a1-bd4f-dd2b45d69b60" />
