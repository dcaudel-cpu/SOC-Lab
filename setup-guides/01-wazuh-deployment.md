# Guide 01: VirtualBox Isolation & Wazuh SIEM Deployment

This guide documents building the foundational homelab infrastructure in **Oracle VirtualBox**, configuring isolated inter-VM networking, deploying an all-in-one **Wazuh SIEM/XDR** server on Kali Linux, and performing an offline agent enrollment for the Windows endpoint (`Win-DC01`).

---

## 🎯 Objectives
* Configure isolated VirtualBox networking (`192.168.56.0/24`) so attack simulations and telemetry remain contained within the lab environment.
* Allocate appropriate compute resources for stable SIEM indexing and endpoint monitoring.
* Deploy the **Wazuh Manager, Indexer, and Dashboard** on Kali Linux.
* Enroll the Windows endpoint offline by hosting the Wazuh Agent MSI installer locally over HTTP.

---

## 🖥️ Step 1: VirtualBox Architecture & Resource Allocation

To keep the Wazuh Indexer and Dashboard responsive while running live endpoint telemetry, resources are split across two virtual machines connected via a **Host-Only / Internal Network** adapter:

| VM Hostname | Role | Operating System | RAM Allocated | Network Configuration |
| :--- | :--- | :--- | :--- | :--- |
| **kali** | SIEM Server & Red Team Node | Kali Linux | 6 GB | Host-Only (`192.168.56.101`) + Temporary NAT (for package downloads) |
| **Win-DC01** | Monitored Endpoint | Windows (x86) | 4 GB | Host-Only (`192.168.56.x`) |

### Snapshot & Isolation Best Practices
* **Network Isolation:** Once core packages are downloaded on Kali, both VMs communicate exclusively over the isolated Host-Only subnet (`192.168.56.0/24`), preventing simulated malicious traffic from touching the host network.
* **Clean Baseline Snapshots:** A fresh snapshot is captured immediately after OS configuration and prior to agent enrollment so the lab can be rolled back cleanly after testing.

---

## 🛡️ Step 2: Install Wazuh All-in-One on Kali Linux

1. **Download and run the official Wazuh installation assistant** on Kali Linux (with temporary NAT enabled for internet access):
   ```bash
   curl -sO https://wazuh.com
   sudo bash ./wazuh-install.sh -a
   ```
2. **Save generated credentials:** Record the initial `admin` password printed at the end of the installation script (or extract it later from `wazuh-install-files.tar`).
3. **Enable Wazuh services to start automatically on boot:**
   ```bash
   sudo systemctl enable wazuh-manager wazuh-indexer wazuh-dashboard
   sudo systemctl status wazuh-manager
   ```
4. **Verify dashboard access:** Open a browser on Kali and navigate to `https://localhost` (or `https://192.168.56.101`), logging in with the `admin` account to confirm the Wazuh web interface loads cleanly.

---

## 📦 Step 3: Offline Windows Agent Enrollment

Because `Win-DC01` is isolated from the external internet, the Wazuh Windows Agent installer (`~14 MB` MSI) is staged on Kali and transferred over the Host-Only network:

1. **Stage and serve the installer on Kali:**
   ```bash
   mkdir ~/wazuh-agent && cd ~/wazuh-agent
   curl -L -o wazuh-agent.msi https://wazuh.com
   python3 -m http.server 8000
   ```
2. **Download and install the agent on Windows:** Open **PowerShell as Administrator** on `Win-DC01` and run:
   ```powershell
   Invoke-WebRequest -Uri "http://192.168.56" -OutFile "C:\wazuh-agent.msi"
   msiexec.exe /i C:\wazuh-agent.msi /q WAZUH_MANAGER="192.168.56.101" WAZUH_AGENT_NAME="Win-DC01"
   ```
3. **Start the Wazuh Agent service:**
   ```powershell
   NET START WazuhSvc
   ```
4. **Verify active connection:** Check the **Agents** tab in the Wazuh Dashboard (or run `sudo /var/ossec/bin/agent_control -l` on Kali) to confirm `Win-DC01` reports as **Active**.

---

## 🔧 Troubleshooting & Lessons Learned

* **Service Startup After VM Reboot:** If `Win-DC01` logs `Unable to connect to ':1514/tcp': No connection could be made because the target machine actively refused it`, verify that `wazuh-manager` is running on Kali (`sudo systemctl start wazuh-manager`) and bound to `0.0.0.0:1514`.
* **Toggling NAT vs. Host-Only Adapters:** Switching Kali to NAT to download external packages changes its active IP route. Always verify `ip -br a` on Kali and confirm `192.168.56.101` is active on the Host-Only interface before testing agent connectivity.

---

## 📸 Verification Screenshots
![Wazuh deployment Screenshot](<../Images/01-wazuh-deployment Screenshot.png>) 
