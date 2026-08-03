---
## 📸 Phase 2 Procedure & Screenshots — Active Response (Automated IP Blocking)

A step-by-step record of implementing automated threat containment, building on the SSH brute-force detection rules established in Phase 1.

---

### 🔷 PHASE 2 — Active Response (Automated IP Blocking)

| Step | Command(s) Used | Screenshot | Description |
|---|---|---|---|
| **1. Enable UFW on Victim (Safely)** | `sudo ufw allow OpenSSH`<br>`sudo ufw enable`<br>`sudo ufw status numbered` | `screenshots/Fig 42.png` | Enabled UFW firewall on Ubuntu-Victim, allowing SSH first to avoid remote lockout, confirmed `Status: active` with SSH permitted on IPv4/IPv6. |
| **2. Locate Wazuh's Active Response Scripts** | `sudo ls -la /var/ossec/active-response/bin/` | `screenshots/Fig 43.png` | Identified the built-in `firewall-drop` script (root:wazuh, 750 permissions) shipped by default with the Wazuh Agent installation. |
| **3. Define Active Response on Manager** | `sudo nano /var/ossec/etc/ossec.conf` *(added `<command>` and `<active-response>` blocks referencing rules 5551, 5710, 5760)* | `screenshots/Fig 44.png` | Configured the Manager to automatically dispatch the `firewall-drop` command to the Agent whenever the SSH brute-force rules from Phase 1 fire, with a 600-second auto-unblock timeout. |
| **4. Validate Config & Restart Manager** | `sudo systemctl restart wazuh-manager`<br>`sudo systemctl status wazuh-manager` | `screenshots/Fig 45.png` | Confirmed clean restart with all modules (`analysisd`, `execd`, `remoted`, etc.) active — no XML parsing errors after correcting two tag-mismatch typos in the config. |
| **5. Trigger the Response** | `./brute_test.sh` *(re-run from Manager)* | `screenshots/Fig 46.png` | Re-ran the Phase 1 brute-force script; observed a live shift in SSH client behavior from `Permission denied` to `Connection timed out` mid-attack — the signature of an active kernel-level block. |
| **6. Capture Live Firewall Block** | `watch -n 1 'sudo iptables -L INPUT -n --line-numbers'` | `screenshots/Fig 47.png` | Live-captured a `DROP` rule for `10.0.2.12` inserted at position 1 of the `INPUT` chain — direct proof of automated containment, independent of UFW's own rule bookkeeping. |
| **7. Confirm Dispatch Trace on Manager** | `sudo grep -i "active-response\|firewall-drop" /var/ossec/logs/ossec.log` | `screenshots/Fig 48.png` | Verified the Manager's JSON dispatch trace: alert → `add` command → Agent `firewall-drop` execution → `check_keys` → `continue` → `Ended`. |
| **8. Verify Auto-Unblock Lifecycle** | *(Dashboard)* **Threat Hunting → Events** → Search: `active-response` | `screenshots/Fig 49.png` | Confirmed **Rule 652** — *"Host Unblocked by firewall-drop Active Response"* (level 3) — fired after the timeout window, proving the full add-block-unblock lifecycle completed and was independently logged. |
| **9. Final Dashboard Summary** | *(Dashboard)* **Threat Hunting → Dashboard** *(no filter, Last 24 hours)* | `screenshots/Fig 50.png` | Full-picture view: 1,378 total events, 34 authentication failures, 441 successes, MITRE ATT&CK breakdown showing Valid Accounts, Password Guessing, Brute Force, SSH, and Network Sniffing techniques covered across testing. |

---

### 📊 Detection Rule Reference Table

| Rule ID | Level | Description | Trigger Condition | MITRE ATT&CK |
|---|---|---|---|---|
| 652 | 3 | Host Unblocked by firewall-drop Active Response | Active-response timeout expiry / auto-unblock | — |

---

*Note: Replace screenshot filenames/numbers above with your actual captured images before committing. This continues the Fig sequence from Phase 1 (Fig 31–41); Phase 2 begins at Fig 42.*
