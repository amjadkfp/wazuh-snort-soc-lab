<div align="center">

# 🛡️ Centralized Threat Detection Through Wazuh–Snort SIEM Integration
### Centralized Intrusion Detection and Monitoring

*A hands-on SOC lab project integrating **Snort** (open-source network IDS) with **Wazuh** (open-source SIEM) to build a centralized threat detection and monitoring pipeline, entirely in a self-hosted virtual lab.*

---

![Wazuh](https://img.shields.io/badge/Wazuh-SIEM-1565C0?style=for-the-badge&logo=wazuh&logoColor=white)
![Snort](https://img.shields.io/badge/Snort-IDS-CC0000?style=for-the-badge&logo=snort&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-Server-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Kali](https://img.shields.io/badge/Kali_Linux-Attacker-557C94?style=for-the-badge&logo=kalilinux&logoColor=white)
![VirtualBox](https://img.shields.io/badge/VirtualBox-Lab-183A61?style=for-the-badge&logo=virtualbox&logoColor=white)
![OpenSearch](https://img.shields.io/badge/OpenSearch-Indexer-005EB8?style=for-the-badge&logo=opensearch&logoColor=white)

![Status](https://img.shields.io/badge/Status-Completed-2ea44f?style=for-the-badge)
![Type](https://img.shields.io/badge/Type-SOC%20Lab-blueviolet?style=for-the-badge)
![Purpose](https://img.shields.io/badge/Purpose-Educational-orange?style=for-the-badge)

</div>

---

## 🗺️ Architecture Diagram

<p align="center">
  <img src="diagrams/architecture_diagram.png" alt="Architecture Diagram" width="100%"/>
</p>

---

## 📋 Overview

This project builds a working pipeline where:

| Step | Component | Action |
|:----:|-----------|--------|
| 1️⃣ | **Snort** | Inspects live network traffic at the packet level and writes a structured alert when a rule matches |
| 2️⃣ | **Wazuh Agent** | Tails Snort's alert log in real time and forwards parsed events |
| 3️⃣ | **Wazuh Manager** | Correlates incoming events and assigns severity / rule metadata |
| 4️⃣ | **Wazuh Dashboard** | Renders the alert for analyst review — searchable, filterable, attributable |

```
Attack traffic → Snort (detection) → Wazuh Agent (forwarding)
→ Wazuh Manager (correlation) → Wazuh Dashboard (visualization)
```

---

## 🖥️ Lab Architecture

| Machine | Role | IP Address | Key Software |
|---------|------|:----------:|--------------|
| 🔵 Wazuh-Manager | SIEM Server | `10.0.2.12` | Wazuh Manager, Dashboard, OpenSearch |
| 🟠 Ubuntu-Victim | Monitored Endpoint | `10.0.2.14` | Snort IDS, Wazuh Agent |
| 🟢 Kali Linux | Planned Attacker Host | `N/A` | Nmap, pentest toolkit |

All VMs were orchestrated in Oracle VirtualBox on a shared NAT Network (`10.0.2.0/24`).

> **⚠️ Note:** Due to host RAM constraints (8 GB), the Kali Linux VM could not run concurrently with the other two machines for the final test. Attack traffic was instead generated from the Wazuh-Manager VM against the victim, preserving the same cross-host network architecture. See the [troubleshooting log](#-troubleshooting-highlights) below for the full reasoning.

---
## 📸 Project Procedure & Screenshots

A step-by-step record of the full build process, from initial VM provisioning through final attack detection and dashboard verification.

| Step | Command(s) Used | Screenshot | Description |
|---|---|---|---|
| **1. Wazuh Manager VM Setup** | *(Wazuh OVA deployed in VirtualBox)* | [![Fig 1](screenshots/Fig%201.png)](screenshots/Fig%201.png) | Deployed the official Wazuh OVA image as a VirtualBox VM. First boot displays default OS login credentials (`wazuh-user` / `wazuh`) for terminal access — separate from the Dashboard's web login. |
| **2. Confirm Wazuh Manager Network Identity** | `ip a` | [![Fig 2](screenshots/Fig%202.png)](screenshots/Fig%202.png) | Verified the Manager's IP inside the VirtualBox NAT Network: `10.0.2.12/24`. This IP becomes the reference point for all agent and attack-traffic configuration. |
| **3. Ubuntu Victim VM Setup** | *(Ubuntu Server 22.04 installed fresh in VirtualBox)* | [![Fig 3](screenshots/Fig%203.png)](screenshots/Fig%203.png) | Installed a standard Ubuntu Server 22.04 VM as the monitored endpoint, to later run Snort (IDS) and the Wazuh Agent. |
| **4. Confirm Ubuntu Victim Network Identity** | `ip a` | [![Fig 4](screenshots/Fig%204.png)](screenshots/Fig%204.png) | Verified the Victim's IP: `10.0.2.14/24`, confirming it sits on the same subnet as the Manager. |
| **5. Kali Linux VM — Planned Attacker** | *(Kali Linux VM created in VirtualBox, 2048MB RAM / 2 vCPU)* | [![Fig 5](screenshots/Fig%205.png)](screenshots/Fig%205.png) | Provisioned a third VM, Kali Linux, as the intended attacker machine for Nmap scans, brute-force attempts, and ping sweeps. |
| **6. Kali Resource Constraint Identified** | Task Manager → Performance → Memory | [![Fig 6](screenshots/Fig%206.png)](screenshots/Fig%206.png) | Booting Kali alongside the other two VMs froze the host. Diagnosed as insufficient host RAM (8GB total) for three simultaneous VMs. |
| **7. Kali RAM/CPU Trimmed (mitigation attempt)** | *(VirtualBox Settings → System, VM powered off)* | [![Fig 7](screenshots/Fig%207.png)](screenshots/Fig%207.png) | Reduced Kali's allocated resources (2048MB→1536MB RAM, 2→1 vCPU) attempting to free enough headroom. |
| **8. Decision: Kali Unused, Pivot to Manager-as-Attacker** | *(architectural decision — no command)* | — | Confirmed via `free -h` that headroom remained negligible even trimmed. Decided to generate attack traffic from the Wazuh-Manager VM instead, since it was already required to run. |
| **9. Import Wazuh GPG Signing Key** | `sudo mkdir -p /usr/share/keyrings`<br>`curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH -o /tmp/wazuh-key.gpg`<br>`sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import /tmp/wazuh-key.gpg`<br>`sudo chmod 644 /usr/share/keyrings/wazuh.gpg` | [![Fig 9](screenshots/Fig%209.png)](screenshots/Fig%209.png) | Imported Wazuh's public signing key into a dedicated keyring on Ubuntu-Victim, required before installing the agent via APT. |
| **10. Add Wazuh APT Repository** | `echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \| sudo tee -a /etc/apt/sources.list.d/wazuh.list`<br>`sudo apt update` | — | Registered Wazuh's package server as a trusted APT source. Confirmed reachable with no signature errors (`Hit:1 ... InRelease`). |
| **11. Install Wazuh Agent with Manager IP Pre-Set** | `sudo WAZUH_MANAGER="10.0.2.12" apt-get install wazuh-agent -y` | — | Installed Wazuh Agent v4.14.5. The `WAZUH_MANAGER` environment variable auto-wrote the Manager's IP into the agent config during install. |
| **12. Verify Agent Configuration** | `sudo cat /var/ossec/etc/ossec.conf \| grep -A 3 "<server>"` | [![Fig 12](screenshots/Fig%2012.png)](screenshots/Fig%2012.png) | Confirmed the agent's config correctly references the Manager's IP, port 1514, and TCP protocol. |
| **13. Enable and Start Wazuh Agent Service** | `sudo systemctl daemon-reload`<br>`sudo systemctl enable wazuh-agent`<br>`sudo systemctl start wazuh-agent`<br>`sudo systemctl status wazuh-agent` | [![Fig 13](screenshots/Fig%2013.png)](screenshots/Fig%2013.png) | Started and enabled the agent service. Confirmed all sub-modules (`wazuh-agentd`, `wazuh-execd`, `wazuh-syscheckd`, `wazuh-logcollector`, `wazuh-modulesd`) launched successfully. |
| **14. Register Agent on Manager** | *(On Manager)* `sudo /var/ossec/bin/manage_agents` → `A` (Add) → `E` (Extract key) | [![Fig 14](screenshots/Fig%2014.png)](screenshots/Fig%2014.png) | Manually registered the Victim agent on the Manager and generated a unique authentication key. |
| **15. Import Key on Victim** | *(On Victim)* `sudo /var/ossec/bin/manage_agents` → `I` (Import) → `y`<br>`sudo systemctl restart wazuh-agent` | [![Fig 15](screenshots/Fig%2015.png)](screenshots/Fig%2015.png) | Imported the key into the Victim's agent config, completing the agent-manager authentication handshake. |
| **16. Verify Agent Connectivity (CLI)** | *(On Manager)* `sudo /var/ossec/bin/agent_control -l` | [![Fig 16](screenshots/Fig%2016.png)](screenshots/Fig%2016.png) | Confirmed agent status `Active` from the Manager's perspective (`ID: 002, Name: ubuntu-victim`). |
| **17. Resolve Dashboard Network Access Issue** | *(VirtualBox NAT Network → Port Forwarding: Host 8443 → Guest 10.0.2.12:443)* | — | Host browser couldn't reach the Manager's internal IP directly (VirtualBox NAT isolates VM IPs from host). Fixed via port forwarding to `https://127.0.0.1:8443`. |
| **18. Verify Agent Connectivity (Dashboard)** | *(Browser)* `https://127.0.0.1:8443` → Overview page | [![Fig 18](screenshots/Fig%2018.png)](screenshots/Fig%2018.png) | Confirmed agent connectivity visually: "Active (1) / Disconnected (0)." Default rules already generating baseline alerts. |
| **19. Locate Snort's Alert Output File** | `sudo find / -iname "*alert*" -path "*snort*" 2>/dev/null`<br>`sudo grep -i "output" /etc/snort/snort.conf \| grep -i alert`<br>`sudo systemctl status snort` | [![Fig 19](screenshots/Fig%2019.png)](screenshots/Fig%2019.png) | Identified Snort writes to three formats. Selected `/var/log/snort/snort.alert.fast` (plain-text) as Wazuh's monitoring target. |
| **20. Add `<localfile>` Directive to Agent Config** | `sudo nano /var/ossec/etc/ossec.conf` *(added `<localfile>` block with `log_format: snort-fast`)* | [![Fig 20](screenshots/Fig%2020.png)](screenshots/Fig%2020.png) | Configured the Wazuh Agent to monitor Snort's alert log, correctly wrapped within its own `<ossec_config>` block. |
| **21. Restart Agent and Confirm Log Monitoring** | `sudo systemctl restart wazuh-agent`<br>`sudo tail -n 20 /var/ossec/logs/ossec.log` | [![Fig 21](screenshots/Fig%2021.png)](screenshots/Fig%2021.png) | Confirmed via agent log: `wazuh-logcollector` actively tailing `/var/log/snort/snort.alert.fast` — the core integration point. |
| **22. Enable Snort's Portscan Detection** | `sudo grep -i "portscan" /etc/snort/snort.conf`<br>`sudo nano /etc/snort/snort.conf` *(uncommented `sfportscan`)*<br>`sudo systemctl restart snort` | — | Found the portscan preprocessor disabled by default. Enabled it and restarted Snort. |
| **23. Diagnose Self-Scan Detection Failure** | `sudo nmap -sS 10.0.2.14` *(from Victim, targeting itself)*<br>`ip route get 10.0.2.14` | [![Fig 23](screenshots/Fig%2023.png)](screenshots/Fig%2023.png) | Self-scans produced zero alerts. Diagnosed via `ip route get`: self-traffic routes through loopback (`lo`), invisible to Snort, which monitors the physical interface only. |
| **24. Generate Genuine Cross-Host Attack Traffic** | *(On Manager)*<br>`sudo yum install nmap -y`<br>`ping -c 4 10.0.2.14`<br>`sudo nmap -sS 10.0.2.14` | [![Fig 24](screenshots/Fig%2024.png)](screenshots/Fig%2024.png) | Pivoted to scanning from the Manager VM — a genuinely separate host — so traffic would cross the real network interface. |
| **25. Confirm Snort Detected the Cross-Host Scan** | *(On Victim)* `sudo tail -n 20 /var/log/snort/snort.alert.fast` | [![Fig 25](screenshots/Fig%2025.png)](screenshots/Fig%2025.png) | Confirmed two fresh, classified SNMP alerts: source `10.0.2.12` → destination `10.0.2.14`, "Attempted Information Leak," Priority 2. |
| **26. Diagnose Dashboard Login Failure** | `sudo systemctl status wazuh-indexer` |  [![Fig 26](screenshots/Fig%2026.png)](screenshots/Fig%2026.png) | Login began failing. Diagnosed `Active: failed (Result: oom-kill)` — OpenSearch killed by the kernel due to VM memory pressure. |
| **27. Restart Indexer and Check Available Memory** | `sudo systemctl start wazuh-indexer`<br>`free -h` | [![Fig 27](screenshots/Fig%2027.png)](screenshots/Fig%2027.png) | Restarted the OpenSearch backend and confirmed sufficient memory before proceeding. |
| **28. Reset Dashboard Admin Password** | `sudo bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin -p 'Soclab.2026' -v` | — | Reset Dashboard credentials with the indexer confirmed healthy (`Clusterstate: GREEN`). |
| **29. Log Into Dashboard with New Credentials** | *(Browser)* `https://127.0.0.1:8443` → `admin` / `Soclab.2026` | [![Fig 29](screenshots/Fig%2029.png)](screenshots/Fig%2029.png) | Confirmed successful login; agent remained connected throughout the outage and recovery. |
| **30. Locate the Snort Alert in the Dashboard** | *(Dashboard)* **Threat Hunting → Events** → Search: `SNMP` | [![Fig 30](screenshots/Fig%2030.png)](screenshots/Fig%2030.png) | Final verification: 2 hits, `agent.name: ubuntu-victim`, `rule.id: 20100`, `rule.level: 8` — full Snort-to-Wazuh-to-Dashboard pipeline confirmed. |

---

## ✅ Result

A live TCP SYN scan launched against the victim host was:

- ✅ Detected by Snort, classified as an **SNMP information-leak attempt** (Priority 2)
- ✅ Forwarded through the Wazuh Agent
- ✅ Correlated by the Wazuh Manager (**Rule ID 20100**, severity level **8**)
- ✅ Rendered as a fully attributed event in the Wazuh Dashboard

> 📁 See `screenshots/` for the full evidence trail, and `docs/Wazuh_Snort_Integration_Report.docx` for the complete write-up.

---

## 🚨 SOC-Style Alert Analysis

<div align="center">

| 🔍 Field | 📄 Detail |
|----------|-----------|
| **Alert Name** | SNMP AgentX/tcp request / SNMP request tcp |
| **Severity** | 🟡 Medium — Wazuh rule level **8** · Snort Priority **2** |
| **Source IP** | `10.0.2.12` |
| **Destination IP** | `10.0.2.14` |
| **MITRE ATT&CK** | `T1046` – Network Service Discovery · `T1040` – Network Sniffing |
| **Impact** | Reconnaissance probing for an exposed SNMP service |
| **Recommended Action** | Disable SNMP if unused · Restrict via firewall · Enforce SNMPv3 |

</div>

---

## 🔧 Troubleshooting Highlights

This project hit (and resolved) several genuine real-world issues — documented in full in the report, summarized here:

<details>
<summary><b>1. 🔑 GPG Keyring Syntax Error</b></summary>

> A single-character typo (`gunpg-ring` vs `gnupg-ring`) caused opaque import failures. Resolved by separating key download from import.

</details>

<details>
<summary><b>2. 🌐 VirtualBox NAT Network Isolation</b></summary>

> The host machine couldn't reach the dashboard directly. Resolved via VirtualBox port forwarding (`127.0.0.1:8443` → `10.0.2.12:443`).

</details>

<details>
<summary><b>3. 👁️ Self-Scan Traffic Invisible to Snort</b></summary>

> Traffic a host sends to its own IP routes through the loopback interface (`lo`), never touching the network adapter Snort monitors. Confirmed via `ip route get`, resolved by generating attack traffic from a genuinely separate host.

</details>

<details>
<summary><b>4. 💥 OpenSearch (Wazuh Indexer) OOM-Kill</b></summary>

> The indexer crashed under host memory pressure, breaking dashboard login. Diagnosed via `systemctl status wazuh-indexer` showing `Result: oom-kill`.

</details>

<details>
<summary><b>5. 🔄 Agent Re-Enrollment Conflict</b></summary>

> A VM reboot caused the agent to register under a new ID, resolved by treating the manager's agent list as the source of truth.

</details>

<details>
<summary><b>6. 🗂️ Malformed <code>ossec.conf</code> XML</b></summary>

> A new `<localfile>` block was added outside any `<ossec_config>` wrapper. Fixed by wrapping it correctly.

</details>

---

## 📁 Repository Structure

```
.
├── 📄 README.md
├── 📂 docs/
│   └── Wazuh_Snort_Integration_Report.pdf   # Full project report
├── 📂 config/
│   ├── ossec.conf.snippet                    # Redacted localfile config (no keys/secrets)
│   └── snort.conf.snippet                    # Redacted sfportscan config
├── 📂 screenshots/                           # Evidence trail, Fig. 1–10
└── 📂 diagrams/
    └── architecture_diagram.png
```

> ⚠️ Configuration files in `config/` are **redacted snippets** for demonstration only. Real deployment files (`client.keys`, full `ossec.conf`) containing authentication secrets are intentionally excluded — see `.gitignore`.

---

## 🛠️ Tools & Technologies

<div align="center">

![Wazuh](https://img.shields.io/badge/Wazuh-1565C0?style=flat-square&logo=wazuh&logoColor=white)
![Snort](https://img.shields.io/badge/Snort-CC0000?style=flat-square&logoColor=white)
![VirtualBox](https://img.shields.io/badge/VirtualBox-183A61?style=flat-square&logo=virtualbox&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu_Server-E95420?style=flat-square&logo=ubuntu&logoColor=white)
![Kali Linux](https://img.shields.io/badge/Kali_Linux-557C94?style=flat-square&logo=kalilinux&logoColor=white)
![OpenSearch](https://img.shields.io/badge/OpenSearch-005EB8?style=flat-square&logo=opensearch&logoColor=white)
![Linux](https://img.shields.io/badge/Linux_CLI-FCC624?style=flat-square&logo=linux&logoColor=black)
![Nmap](https://img.shields.io/badge/Nmap-4B0082?style=flat-square&logoColor=white)

</div>

---

## 📜 License

> This project is for educational purposes as part of a SOC / cybersecurity coursework lab.

---

<div align="center">

*Built with 🔐 for learning — by a future SOC analyst.*

</div>
