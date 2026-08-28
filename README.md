<div align="center">

# 🛡️ Wazuh–Snort SOC Lab: SIEM Detection, SOAR-Style Response, Custom Rule Engineering and False Positive Reduction
### SOAR-Style Response · SIEM Detection · File Integrity Monitoring

*A six-phase, hands-on SOC engineering project — from raw network intrusion detection to custom-authored SIEM correlation rules and formal alert tuning — built entirely in a self-hosted virtual lab.*

---

![Wazuh](https://img.shields.io/badge/Wazuh-SIEM-1565C0?style=for-the-badge&logo=wazuh&logoColor=white)
![Snort](https://img.shields.io/badge/Snort-IDS-CC0000?style=for-the-badge&logo=snort&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-Server-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Kali](https://img.shields.io/badge/Kali_Linux-Attacker-557C94?style=for-the-badge&logo=kalilinux&logoColor=white)
![VirtualBox](https://img.shields.io/badge/VirtualBox-Lab-183A61?style=for-the-badge&logo=virtualbox&logoColor=white)
![OpenSearch](https://img.shields.io/badge/OpenSearch-Indexer-005EB8?style=for-the-badge&logo=opensearch&logoColor=white)

![Status](https://img.shields.io/badge/Status-Completed-2ea44f?style=for-the-badge)
![Phases](https://img.shields.io/badge/Phases-5%20Completed%20%2B%201%20Planned-blueviolet?style=for-the-badge)
![Purpose](https://img.shields.io/badge/Purpose-Educational-orange?style=for-the-badge)

</div>

---

> ### ⚠️ A note on "SOAR"
> This project is **not** a full SOAR platform. What it *does* demonstrate is the fundamental SOAR concept — **automated, detection-triggered response** — implemented using Wazuh's native Active Response module. A production SOAR would layer on cross-tool orchestration, ticketing integration, and analyst approval workflows on top of what's built here. This project is scoped and honest about that boundary — and that's deliberate.

---

## 🗺️ Overall Architecture

<p align="center">
  <img src="diagrams/overall_system_architecture.png" alt="Overall System Architecture" width="100%"/>
</p>

<div align="center">

```
Attack / Log Event → Detection (Snort + Wazuh Rules) → Correlation (Wazuh Manager)
      → Response (Active Response) → Visualization (Wazuh Dashboard) → Tuning & Validation
```

</div>

---

## 📚 Project Phases

Each phase builds directly on the last — same lab, same agent, same Manager — progressively layering detection, response, integrity monitoring, custom engineering, and formal tuning on top of one shared SOC pipeline.

<table>
<tr>
<td width="8%" align="center"><b>0</b></td>
<td width="42%">🌐 <b>Centralized Threat Detection</b><br><sub>Snort IDS + Wazuh SIEM Integration</sub></td>
<td width="30%">Network IDS-to-SIEM pipeline</td>
<td width="20%" align="center"><a href="docs/Phase0_Snort_IDS_Integration.md"><b>View →</b></a></td>
</tr>
<tr>
<td align="center"><b>1</b></td>
<td>🔐 <b>Host-Based Intrusion Detection</b><br><sub>SSH Brute-Force Log Analysis</sub></td>
<td>Log-based correlation rules</td>
<td align="center"><a href="docs/Phase1_SSH_BruteForce_Detection.md"><b>View →</b></a></td>
</tr>
<tr>
<td align="center"><b>2</b></td>
<td>⚡ <b>Automated Threat Containment</b><br><sub>Wazuh Active Response</sub></td>
<td>SOAR-style auto-remediation</td>
<td align="center"><a href="docs/Phase2_Active_Response.md"><b>View →</b></a></td>
</tr>
<tr>
<td align="center"><b>3</b></td>
<td>🗂️ <b>Endpoint Integrity Assurance</b><br><sub>Real-Time File Integrity Monitoring</sub></td>
<td>FIM, hash-diff verification</td>
<td align="center"><a href="docs/Phase3_File_Integrity_Monitoring.md"><b>View →</b></a></td>
</tr>
<tr>
<td align="center"><b>4</b></td>
<td>🧬 <b>Custom Detection Engineering</b><br><sub>Wazuh Rule Authoring</sub></td>
<td>Writing & debugging SIEM correlation rules</td>
<td align="center"><a href="docs/Phase4_Custom_Rule_Authoring.md"><b>View →</b></a></td>
</tr>
<tr>
<td align="center"><b>5</b></td>
<td>🎯 <b>Alert Tuning & False Positive Reduction</b><br><sub>Detection Engineering — Reducing Alert Fatigue</sub></td>
<td>Baselining, classification, rule-level tuning, regression testing</td>
<td align="center"><a href="docs/Phase5_Alert_Tuning_False_Positive_Reduction.md"><b>View →</b></a></td>
</tr>
<tr>
<td align="center"><b>6</b></td>
<td>🔗 <b>Workflow Automation</b> <sub>(Planned)</sub><br><sub>n8n-Orchestrated Incident Response</sub></td>
<td>SOAR orchestration layer</td>
<td align="center"><sub><i>Coming soon</i></sub></td>
</tr>
</table>

> 🔜 **Phase 6 (Planned):** Add [n8n](https://n8n.io) as a workflow automation layer on top of the existing, now-tuned Wazuh pipeline — consuming Wazuh alerts via webhook, automating investigation steps (IP enrichment, threat intel lookups), and orchestrating incident response actions across tools. This is the piece that would move this project from "SOAR-style response" (single-tool automated remediation) toward genuine SOAR orchestration (cross-tool workflows).

---

## 🖥️ Lab Architecture

<div align="center">

| Machine | Role | IP Address | Key Software |
|:---:|:---|:---:|:---|
| 🔵 **Wazuh-Manager** | SIEM Server | `10.0.2.12` | Wazuh Manager, Dashboard, OpenSearch |
| 🟠 **Ubuntu-Victim** | Monitored Endpoint | `10.0.2.14` | Snort IDS, Wazuh Agent |
| 🟢 **Kali Linux** | Attacker Host | `variable` | Nmap, SSH brute-force tooling |

</div>

All VMs run in Oracle VirtualBox on a shared NAT Network (`10.0.2.0/24`). Where host resource constraints limited concurrent VM uptime, attack traffic was generated from an alternate cross-host source — documented transparently per phase rather than silently substituted.

---

## 🔧 Notable Troubleshooting

Real SOC engineering rarely works on the first attempt. These are the debugging stories that best demonstrate hands-on problem-solving across all six phases — full detail lives in each phase's doc.

<details>
<summary><b>🔑 GPG keyring import failure — single-character typo</b></summary>
<br>

> A typo (`gunpg-ring` vs `gnupg-ring`) caused opaque, unhelpful APT signature errors during Wazuh Agent installation. Resolved by separating key download from import and testing each step independently.

</details>

<details>
<summary><b>🌐 VirtualBox NAT isolation blocking Dashboard access</b></summary>
<br>

> The host browser couldn't reach the Manager's internal VM IP directly. Resolved via VirtualBox port forwarding, mapping `127.0.0.1:8443` on the host to `10.0.2.12:443` on the guest.

</details>

<details>
<summary><b>👁️ Self-scan traffic invisible to Snort</b></summary>
<br>

> A host scanning its own IP routes traffic through the loopback interface (`lo`), never touching the physical adapter Snort monitors. Confirmed via `ip route get`, resolved by generating attack traffic from a genuinely separate host.

</details>

<details>
<summary><b>💥 OpenSearch (Wazuh Indexer) OOM-kill under memory pressure</b></summary>
<br>

> The indexer was silently killed by the kernel, breaking Dashboard login with no obvious error at first glance. Root-caused via `systemctl status wazuh-indexer` showing `Result: oom-kill`.

</details>

<details>
<summary><b>🧩 XML parser reporting a misleading error line</b></summary>
<br>

> A custom rule's config failed to load with `XMLERR: Attribute '&lt;if_sid' has no value`, pointing at a line that looked syntactically fine. Root cause: an unclosed `&lt;rule&gt;` tag one line above — the parser was still "inside" the open tag when it read the next line. A reminder that reported line numbers in multi-tag XML files can be one or two lines off from the real defect.

</details>

<details>
<summary><b>🎯 Custom rule silently pre-empted by a more specific sibling rule</b></summary>
<br>

> A custom detection rule failed to fire even though its target log line was confirmed present in the alert stream. Root cause: the rule was parented to an overly generic base rule (`if_sid 1002`), while the event was already being classified — and correlation halted — under a more specific sibling rule (`5402`). Fixed by reparenting to the correct, more specific base rule. This reflects a core lesson in Wazuh rule design: correlation is a tree, not a flat list, and the wrong parent silently starves a custom rule of the events it's meant to catch.

</details>

<details>
<summary><b>🚫 Active Response self-lockout during Phase 5 troubleshooting</b></summary>
<br>

> While diagnosing an unrelated SSH configuration issue, repeated invalid-username login attempts from the Wazuh Manager's own IP triggered rule `5710`, and Phase 2's `firewall-drop` Active Response auto-blocked the Manager itself. Resolved by manually clearing the `iptables` rule on the Victim. A genuine SOC finding: aggressive automated response can create administrative friction against trusted infrastructure — full writeup in the Phase 5 doc.

</details>

<details>
<summary><b>📉 Tuned rule matched but never appeared in the Dashboard (Phase 5)</b></summary>
<br>

> A new tuning rule set to `level="2"` matched correctly but never showed up in Discover. Root cause: Wazuh's default `<log_alert_level>3</log_alert_level>` silently drops any rule below level 3 from being indexed at all, regardless of whether it matched. Resolved by raising the rule to `level="3"`.

</details>

---

## 🚨 SOC-Style Alert Reference (sample)

<div align="center">

| 🔍 Field | 📄 Detail |
|:---|:---|
| **Alert Name** | SNMP AgentX/tcp request — Attempted Information Leak |
| **Severity** | 🟡 Medium — Wazuh Level **8** · Snort Priority **2** |
| **Source → Destination** | `10.0.2.12` → `10.0.2.14` |
| **MITRE ATT&CK** | `T1046` Network Service Discovery · `T1040` Network Sniffing |
| **Impact** | Reconnaissance probing for an exposed SNMP service |
| **Recommended Action** | Disable SNMP if unused · Restrict via firewall · Enforce SNMPv3 |

</div>

> Full SOC analysis tables (Alert Name, Severity, Source/Destination, MITRE ATT&CK, Impact, Recommended Action) for every detection across all six phases are documented in each phase's `docs/` file.

---

## 📁 Repository Structure

```
wazuh-snort-soc-lab/
├── 📄 README.md                        ← You are here
│
├── 📂 docs/
│   ├── Phase0_Snort_IDS_Integration.md
│   ├── Phase1_SSH_BruteForce_Detection.md
│   ├── Phase2_Active_Response.md
│   ├── Phase3_File_Integrity_Monitoring.md
│   ├── Phase4_Custom_Rule_Authoring.md
│   ├── Phase5_Alert_Tuning_False_Positive_Reduction.md
│   └── Final_Report.pdf                ← Consolidated writeup
│
├── 📂 diagrams/
│   ├── overall_system_architecture.png
│   ├── phase0_architecture.png
│   ├── phase1_architecture.png
│   ├── phase2_architecture.png
│   ├── phase3_architecture.png
│   ├── phase4_architecture.png
│   └── phase5_architecture.png
│
├── 📂 screenshots/
│   ├── phase0/
│   ├── phase1/
│   ├── phase2/
│   ├── phase3/
│   ├── phase4/
│   └── phase5/
│
├── 📂 config/
│   ├── local_rules.xml                 # Redacted — no secrets
│   ├── ossec.conf.snippet
│   └── snort.conf.snippet
│
└── 📄 LICENSE
```

> ⚠️ Files in `config/` are **redacted snippets** for demonstration only. Files containing authentication secrets (`client.keys`, full `ossec.conf`) are intentionally excluded — see `.gitignore`.

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
![XML](https://img.shields.io/badge/XML-Rule_Authoring-F26522?style=flat-square&logo=xml&logoColor=white)

</div>

---

## 📜 License

> This project is for educational purposes as part of independent SOC / cybersecurity coursework.

---

<div align="center">

*Built with 🔐 for learning — six phases, one SOC pipeline, zero shortcuts.*

</div>
