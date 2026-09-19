<div align="center">

# Phase 6 - n8n SOAR Orchestration Layer
### Workflow Automation for Wazuh Alert Enrichment & Notification

*A step-by-step, in-progress record of standing up a self-hosted n8n automation layer on top of the existing Wazuh-Snort pipeline, exposing it securely to the internet without a VPS or open inbound ports, and wiring it to Wazuh's native Active Response module for genuine automated triggering.*

---

![n8n](https://img.shields.io/badge/n8n-Workflow_Automation-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-Tunnel-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-20.x_LTS-339933?style=for-the-badge&logo=node.js&logoColor=white)
![Wazuh](https://img.shields.io/badge/Wazuh-SIEM-1565C0?style=for-the-badge&logo=wazuh&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-Server-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)

![Status](https://img.shields.io/badge/Status-In_Progress-yellow?style=for-the-badge)
![Type](https://img.shields.io/badge/Type-SOC_Lab-blueviolet?style=for-the-badge)
![Purpose](https://img.shields.io/badge/Purpose-Educational-orange?style=for-the-badge)

</div>

---

## 🎯 Objective

Phases 0–5 built a complete detection, response, and tuning pipeline entirely within Wazuh + Snort. Phase 6 adds a genuine **SOAR orchestration layer** on top: when a high-severity Wazuh alert fires (rule `5710`/`100010`, SSH brute-force), an active-response script fires a webhook that triggers an [n8n](https://n8n.io) workflow that:

1. Receives and authenticates the alert payload
2. Enriches the source IP (geolocation via `ip-api.com`)
3. Formats a readable, analyst-friendly summary
4. Pushes a real-time notification to Discord

This closes the loop from *detection* → *automated triage support* → *human notification*, the core concept behind SOAR (Security Orchestration, Automation, and Response), while staying honest about the difference between this single-tool automation and a full cross-tool SOAR platform (see README's note on scope).

---

## 🖥️ Infrastructure Decision: Why No Third VM / No Cloud VPS

**Constraint:** 8GB RAM host, 2 existing VMs (Wazuh Manager, Ubuntu Victim) already running Snort + Wazuh Agent. A prior Wazuh Indexer OOM-kill incident (documented in Phase 0/1) made the team deliberately cautious about adding load.

**Options evaluated:**

| Option | Verdict |
|---|---|
| Docker Desktop / third local VM | ❌ Ruled out — insufficient host RAM headroom |
| Oracle Cloud Always Free (Ampere A1) | ❌ Ruled out — requires a credit card for identity verification, even though never billed; not available without one |
| n8n Cloud SaaS | ❌ Trial-only, not viable long-term for a portfolio project |
| Render / Railway free tier | ⚠️ Workable but cold-start delays undermine real-time alerting |
| **n8n self-hosted on existing Ubuntu Victim VM + Cloudflare Tunnel** | ✅ **Selected** |

**Rationale:** n8n installed via `npm` (not Docker) has a small footprint (~200Mi resident RAM measured in practice — see Memory Impact table below). Combined with Cloudflare Tunnel's **fully outbound-initiated** connection model, this avoids opening any inbound port on the lab network entirely — a stronger security posture than a traditional VPS + open firewall port, and a genuine architectural decision worth defending in review: *no listening port exists for an attacker to discover or scan*.

---

## 📊 Memory Impact — Empirically Validated

Given the prior OOM-kill history, every install step was memory-checked before and after.

<div align="center">

| Stage | Available RAM (`free -h`) |
|:---|:---:|
| Baseline (Snort + Wazuh Agent only, post security patch + reboot) | 967Mi |
| After Node.js 20.x install | 985Mi |
| After n8n install (idle, not running) | 1.0Gi |
| **With n8n actively running** | **767Mi** |
| Swap usage under load | 87Mi (of 2.1Gi — untouched majority) |

</div>

**n8n's real-world memory cost: ~200Mi.** Comfortably within budget, with ~750Mi headroom remaining and a full swap safety margin unused. n8n 2.x's process model was also observed and documented: it splits into a main process plus an isolated **task-runner subprocess** for workflow execution sandboxing — a defense-in-depth detail worth noting.

<p align="center">
  <img src="../screenshots/phase6/01_memory_baseline_free_h.png" alt="Memory baseline before n8n install" width="80%"/>
</p>

> ⚠️ **Note (added during Active Response integration):** the memory picture above reflects Ubuntu-Victim only. During live Active Response testing, the **Wazuh-Manager VM** was separately found to be under real memory pressure (see Troubleshooting below) — the two VMs' memory headroom is not symmetric, and this became a genuine testing bottleneck, not just a theoretical risk.

---

## 🔧 Build Log

### Step 1 — Security Patching (pre-work)

Before exposing any new service to the internet, the Ubuntu Victim VM's pending security updates were applied via a **dry-run-then-apply** process (`apt-get -s dist-upgrade` reviewed before real execution), specifically checking that no patch touched `snort` or `wazuh-agent` package versions. A kernel update was included (`5.15.0-185` → `5.15.0-191`), requiring a reboot; Snort and Wazuh Agent service health was verified both immediately pre-reboot and post-reboot to confirm no regression from the kernel bump.

### Step 2 — Node.js 20.x LTS Installation

Installed via the NodeSource repository (required — Ubuntu 22.04's default `apt` repo only carries Node 12.x, which is EOL and incompatible with n8n).

<p align="center">
  <img src="../screenshots/phase6/02_nodejs_correct_version_installed.png" alt="Node.js 20.x correctly installed" width="70%"/>
</p>

### Step 3 — n8n Installation

```bash
sudo npm install -g n8n
```

Version installed: **2.8.4**. Install took ~17 minutes on this VM's constrained CPU allocation (1917 packages), a useful data point for planning future installs on similarly constrained hardware.

<p align="center">
  <img src="../screenshots/phase6/03_n8n_running_http_200_check.png" alt="n8n running, HTTP 200 health check" width="70%"/>
</p>

### Step 4 — Cloudflare Tunnel Installation & Configuration

Installed via Cloudflare's official apt repository (`pkg.cloudflare.com`), version **2026.8.3**. Runs in **Quick Tunnel** mode (`cloudflared tunnel --url http://localhost:5678 --protocol http2`) — no Cloudflare account required, generates a random public `*.trycloudflare.com` HTTPS URL with a fully outbound connection to Cloudflare's edge.

**Known limitation (tracked, not yet resolved):** Quick Tunnel URLs are ephemeral — a new random URL is issued every time the `cloudflared` process restarts, requiring the service to be manually restarted each session (see Session Restart Procedure below). A **Named Tunnel** (requires free Cloudflare account + a domain) is the planned upgrade path for a persistent, production-style URL. Deferred as a later task since it doesn't block current build/test work.

<p align="center">
  <img src="../screenshots/phase6/04_cloudflare_tunnel_first_success.png" alt="Cloudflare Quick Tunnel successfully registered" width="80%"/>
</p>

A significant mid-build obstacle was a broken DHCP-assigned DNS server preventing tunnel registration entirely (full root-cause narrative in Troubleshooting below); the permanent fix via a custom netplan override is confirmed working here:

<p align="center">
  <img src="../screenshots/phase6/05_netplan_dns_fix_confirmed.png" alt="DNS fix confirmed via resolvectl status" width="70%"/>
</p>

### Step 5 — n8n Webhook Node (Trigger)

End-to-end reachability was confirmed first by loading the n8n editor itself through the public tunnel URL from an external browser (i.e., genuinely over the internet, not just `localhost`):

<p align="center">
  <img src="../screenshots/phase6/06_n8n_editor_loaded_via_public_url.png" alt="n8n editor loaded via public Cloudflare Tunnel URL" width="85%"/>
</p>

Configured as the workflow's entry point:

| Setting | Value | Rationale |
|---|---|---|
| HTTP Method | `POST` | Matches how Wazuh's Active Response script delivers alert JSON |
| Path | Random UUID (n8n auto-generated, kept as-is) | Acts as a lightweight secret in the URL — reduces discoverability |
| Authentication | Header Auth (`X-Wazuh-Secret` + 64-char random hex secret via `openssl rand -hex 32`) | Real access control — requests without the correct header/value are rejected |
| Respond | Immediately | Keeps the caller (Wazuh) non-blocked while downstream enrichment/notification runs |

**Verified working** via a manual `curl` test simulating a Wazuh rule `5710` alert payload — the webhook correctly received and authenticated the request, capturing `rule.id`, `rule.description`, and `srcip` in the payload body.

<p align="center">
  <img src="../screenshots/phase6/07_webhook_node_post_header_auth_configured.png" alt="Webhook node configured: POST, random path, Header Auth" width="85%"/>
  <img src="../screenshots/phase6/08_webhook_captured_test_payload_with_secret.png" alt="Webhook captured test payload including X-Wazuh-Secret header" width="85%"/>
</p>

**Critical discovery — Test URL vs Production URL:** n8n exposes two distinct webhook endpoints per node — a `/webhook-test/...` path that only listens while "Listen for test event" is actively clicked in the editor, and a `/webhook/...` **Production URL** that listens permanently once the workflow is **Published**. This n8n version (2.8.4) does not have a separate Active/Inactive toggle — clicking **Publish** is itself the activation mechanism. This distinction matters enormously for Wazuh integration: Active Response scripts cannot rely on someone having the editor open and "listening" — they need the always-on Production URL.

### Step 6 — IP Enrichment Node (HTTP Request)

Added an HTTP Request node calling `ip-api.com` (free tier, no API key required) with the source IP dynamically extracted from the incoming webhook payload via the n8n expression `{{ $json.body.srcip }}`.

**Initial test** — executed against the placeholder test IP `192.168.1.100`, correctly returning `status: fail, message: private range` from ip-api.com — the *correct* expected result for a private/RFC 1918 test address, confirming the request mechanism itself (URL construction, expression resolution, live HTTP call) functions correctly.

<p align="center">
  <img src="../screenshots/phase6/09_ip_enrichment_http_request_success.png" alt="ip-api.com enrichment request correctly resolved and executed" width="85%"/>
</p>

**✅ Real-IP validation (completed):** re-tested against a genuine public IP (`8.8.8.8`) with **zero interaction with the n8n editor UI** — the request was fired at the Production URL directly via `curl` from the terminal, proving the webhook is genuinely backend-listening rather than only working in a UI demo. Real geolocation/ISP data returned correctly:

status: success
country: United States
region: VA
city: Ashburn
isp: Google LLC
as: AS15169 Google LLC


<p align="center">
  <img src="../screenshots/phase6/06_n8n_webhook_production_url_confirmed.png" alt="n8n Webhook node Production URL tab confirmed" width="85%"/>
</p>

### Step 7 — Message Formatting Node (Edit Fields / Set)

Added an Edit Fields node after the HTTP Request node to assemble a single, human-readable `alert_message` string combining data from **two different upstream nodes** in the chain — the original Webhook payload (`rule.id`, `rule.description`, `srcip`) and the HTTP Request enrichment output (`country`, `city`, `isp`), using n8n's cross-node reference syntax `$('Webhook').item.json...` to reach back past the immediately-preceding node.

Fallback logic (`{{ $json.country || 'Unknown' }}`) was added so the message degrades gracefully to "Unknown"/"N/A" when enrichment data is unavailable (as with private/test IPs), rather than rendering broken template syntax or blank fields.

**Verified working** — full message correctly resolved with real enrichment data:

🚨 Wazuh Security Alert
Rule ID: 5710
Description: sshd brute force test
Source IP: 8.8.8.8
Location: United States, Ashburn
ISP: Google LLC
⚠️ Review and respond if necessary.


<p align="center">
  <img src="../screenshots/phase6/10_edit_fields_resolved_alert_message.png" alt="Edit Fields node fully resolved alert_message combining two upstream nodes" width="85%"/>
</p>

### Step 8 — Discord Notification Node

Added a Discord node in **Webhook connection mode** (rather than Bot Token, which requires a full Discord Developer Portal application + bot invite flow — unnecessary complexity for this use case). Configured with `Operation: Send a Message`, message body bound to `{{ $json.alert_message }}` from the prior Edit Fields node.

**Verified working end-to-end** — a real message was posted to the `#wazuh-alert` Discord channel by "Wazuh SOAR Bot," rendering full Markdown formatting (bold field labels, emoji) correctly.

<p align="center">
  <img src="../screenshots/phase6/11_discord_node_execution_success.png" alt="Discord node executed successfully" width="85%"/>
</p>

**✅ Real-IP end-to-end proof** — fired purely at the Production URL, no editor interaction, real public-IP enrichment data landing correctly formatted in Discord:

<p align="center">
  <img src="../screenshots/phase6/07_discord_final_alert_real_geolocation_delivered.png" alt="Final Wazuh Security Alert message with real geolocation delivered to Discord channel via Production URL" width="85%"/>
</p>

### Step 9 — Wazuh Active Response Integration (In Progress)

With the n8n pipeline fully validated as an always-on backend service, the remaining piece is making **Wazuh itself** call the Production URL automatically — no `curl` run by hand.

**9.1 — Custom Active Response script.** Wazuh's Active Response dispatches a JSON alert payload via stdin to a script placed in `/var/ossec/active-response/bin/` on the agent, matching the exact ownership/permission convention of the built-in `firewall-drop` script from Phase 2 (`root:wazuh`, mode `750`):

```bash
sudo chown root:wazuh /var/ossec/active-response/bin/n8n-notify-debug.sh
sudo chmod 750 /var/ossec/active-response/bin/n8n-notify-debug.sh
```

<p align="center">
  <img src="../screenshots/phase6/08_active_response_bin_permissions_jq_installed.png" alt="jq installed, active-response bin directory permissions convention confirmed (firewall-drop, wazuh-slack, etc.)" width="85%"/>
</p>

A **diagnostic version** of the script (`n8n-notify-debug.sh`) was written first, deliberately deferring the final `jq`-based JSON parsing logic until the real Wazuh stdin payload structure could be observed directly, rather than guessing field paths:

```bash
#!/bin/bash
echo "--- New trigger at $(date) ---" >> /tmp/wazuh-ar-debug.log
cat >> /tmp/wazuh-ar-debug.log
exit 0
```

Manually verified working before wiring into Wazuh at all:
```bash
echo '{"test":"manual invocation"}' | sudo /var/ossec/active-response/bin/n8n-notify-debug.sh
cat /tmp/wazuh-ar-debug.log
```

<p align="center">
  <img src="../screenshots/phase6/09_n8n_notify_debug_script_deployed_correct_perms.png" alt="n8n-notify-debug.sh deployed with correct root:wazuh 750 permissions" width="85%"/>
</p>

**9.2 — Wazuh-Manager configuration.** Following the exact pattern of Phase 2's `firewall-drop` binding (backed up first as `ossec.conf.bak-phase6`), a new `<command>` and `<active-response>` block pair was appended to the Manager's `ossec.conf`, bound to custom correlation rule `100010` (rather than the noisier raw rule `5710`) to keep the notification signal clean — one alert per confirmed brute-force pattern, not one per failed login attempt:

<p align="center">
  <img src="../screenshots/phase6/10_manager_existing_firewall_drop_active_response_block.png" alt="Located the real, working firewall-drop active-response binding on the Manager as the template to follow" width="85%"/>
</p>

```xml
<command>
  <name>n8n-notify</name>
  <executable>n8n-notify-debug.sh</executable>
  <timeout_allowed>no</timeout_allowed>
</command>

<active-response>
  <disabled>no</disabled>
  <command>n8n-notify</command>
  <location>local</location>
  <rules_id>100010</rules_id>
</active-response>
```

Config validated clean (`wazuh-analysisd -t`, no errors) and the Manager restarted successfully with all modules up:

<p align="center">
  <img src="../screenshots/phase6/11_manager_restart_clean_no_xml_errors.png" alt="Manager restart clean, no XML parsing errors, all modules started" width="85%"/>
  <img src="../screenshots/phase6/12_manager_restart_with_new_n8n_notify_block.png" alt="Manager config reload confirmed with new n8n-notify active-response block present" width="85%"/>
</p>

**9.3 — Live trigger test.** Rule `100010` (and its underlying `5710`) confirmed firing correctly via repeated SSH brute-force attempts:

<p align="center">
  <img src="../screenshots/phase6/13_rule_100010_and_5710_firing_confirmed.png" alt="Rule 100010 and 5710 confirmed firing in alerts.log" width="85%"/>
</p>

However — **the active-response dispatch itself never fired**, despite the rule correctly matching. `/tmp/wazuh-ar-debug.log` remained empty/nonexistent after multiple trigger attempts, and no execution trace appeared in either VM's `ossec.log`:

<p align="center">
  <img src="../screenshots/phase6/14_debug_log_missing_dispatch_never_happened.png" alt="Debug log file never created despite rule 100010 firing — dispatch silently not happening" width="85%"/>
</p>

Full root-cause investigation and current status documented in Troubleshooting below — **this is the active blocker for Phase 6 completion.**

---

## 🧰 Notable Troubleshooting (real incidents, documented in full)

<details>
<summary><b>⌨️ Repeated DNS typo (deb.nodesource.com → ded.nodesource.com)</b></summary>
<br>

> The NodeSource install script failed silently due to a one-character DNS typo, causing `apt-get install nodejs` to fall back to Ubuntu's stale default repo (Node 12.x) without any obvious error at the install step itself. Caught only by explicitly checking `node -v` after install rather than assuming success. Root-caused, corrected, and re-verified across two full retry cycles before the correct NodeSource repo was properly registered.

</details>

<details>
<summary><b>🌐 Cloudflare Tunnel registration hanging indefinitely (root cause: broken DHCP-assigned DNS)</b></summary>
<br>

> `cloudflared tunnel` would pass all environment preflight checks (DNS resolution, TCP/UDP connectivity, Cloudflare API reachability) but then hang indefinitely at tunnel registration, even after forcing HTTP2 over QUIC. Initially suspected to be mobile-carrier deep packet inspection (a genuine and common issue on Indian mobile networks) after switching hotspots produced a *different*, more specific error: `dial tcp: lookup api.trycloudflare.com on 127.0.0.1:53: i/o timeout`.
>
> Root-caused to the VM's DHCP-assigned DNS server (`192.168.103.185`) being unreachable through VirtualBox's NAT layer, despite general internet connectivity (`ping 1.1.1.1`) working fine. Runtime fixes via `resolvectl dns` were repeatedly and silently overridden by DHCP re-injection. Permanently resolved by identifying that `/etc/netplan/50-cloud-init.yaml` is cloud-init-managed (edits don't persist across reboot) and instead creating a separate override file `/etc/netplan/99-custom-dns.yaml` with explicit `dhcp4-overrides: use-dns: false` to fully suppress DHCP-supplied DNS in favor of `1.1.1.1`/`8.8.8.8`. A secondary file-permission issue (`600` required, initially `644`) was also caught and corrected along the way.
>
> **Lesson:** a hung network handshake can have a root cause several layers removed from the failing tool itself — proper diagnosis required peeling back from application-layer symptom (tunnel timeout) → protocol test (QUIC vs HTTP2) → raw connectivity test → DNS-specific test → persistent-vs-runtime config distinction, rather than accepting the first plausible explanation (carrier DPI).

</details>

<details>
<summary><b>🖥️ VirtualBox console keyboard-shortcut interception blocking tmux</b></summary>
<br>

> Attempted to use `tmux` to split a single terminal into two panes (to monitor `free -h` while `n8n start` ran in the foreground) for lack of a working second SSH path from the Windows host (no port-forward configured for SSH, unlike the Wazuh Dashboard's existing port-forward). `Ctrl+B` prefix commands were silently swallowed by the VirtualBox console window rather than reaching tmux, evidenced by the literal characters appearing typed at the shell prompt instead of triggering a pane split. Resolved by abandoning tmux for this use case and instead backgrounding long-running processes properly (`Ctrl+Z` → `bg` → `disown -h`) combined with `nohup ... &` for services that need to persist independent of any single terminal session.

</details>

<details>
<summary><b>📋 Multi-line curl commands corrupted by VM console paste handling</b></summary>
<br>

> Multiple attempts to paste a multi-line `curl` command (using `\` line continuations) directly into the VirtualBox console terminal resulted in corrupted commands — dropped characters (notably the `5678` port number, `/` path separators, and spaces between arguments vanished from pasted content on several separate occasions), producing confusing "bad/illegal URL format," "nested brace," and "bad configuration option" errors that did not reflect any actual mistake in the command as written. Resolved by writing commands into a file via `nano` and executing as a script (`bash script.sh`) rather than pasting/typing directly at the shell prompt. **Lesson:** when a command that looks syntactically correct produces bizarre parser errors, verify the *actual* received input (`cat` the file) before assuming a logic error — the terminal's input-handling path itself was repeatedly the fault, not the command's logic. This became a recurring, load-bearing lesson throughout Phase 6, not a one-off.

</details>

<details>
<summary><b>🔗 n8n test-node execution requires re-priming the full upstream chain after a session gap</b></summary>
<br>

> Returning to the workflow in a new session, the Edit Fields node's "Execute step" initially failed silently ("No output data") because n8n's per-node test cache does not automatically persist/propagate across nodes when reopening a workflow — each node in the chain (Webhook → HTTP Request → Edit Fields) needed to be re-triggered in order (re-listen + re-send test webhook, then re-execute HTTP Request, then re-execute Edit Fields) before the final node had valid upstream data to resolve its cross-node expressions against. n8n's own in-editor hint (`Tip: Execute previous nodes to use input data`) pointed directly at the fix once noticed.

</details>

<details>
<summary><b>🌐 Discord webhook returning HTTP 301 on first credential attempt</b></summary>
<br>

> The first Discord Webhook credential produced `301 - ""` on execution — an unexpected redirect response where Discord's webhook API normally returns 200/204 on success. Root cause not conclusively isolated (candidates: a stray character/whitespace introduced during copy-paste from Discord's webhook URL, or a request to the legacy `discordapp.com` domain rather than the current `discord.com`), but resolved cleanly by deleting and re-creating the credential with a freshly re-copied URL, which then tested and executed successfully (`success: true`, message confirmed delivered to Discord). **Lesson:** for opaque low-level HTTP errors on a third-party webhook, re-issuing the credential from a clean copy is often faster than exhaustively diagnosing the exact byte-level cause.

</details>

<details>
<summary><b>🔑 Test URL vs Production URL — the webhook only fires standalone once Published</b></summary>
<br>

> Early validation only ever used n8n's `/webhook-test/...` URL, which requires the editor's "Listen for test event" to be actively clicked — meaning the pipeline appeared to work, but only while someone was babysitting the UI. This would have been silently useless for real Wazuh integration. Root-caused by carefully distinguishing the Test URL tab from the Production URL tab on the Webhook node, and discovering this n8n version's **Publish** button (not a separate Active/Inactive toggle) is what makes the Production URL listen permanently. Confirmed fixed by firing a `curl` POST at the Production URL with zero interaction with the n8n editor and receiving a correct, fully-enriched Discord alert — proof the backend listener genuinely works standalone.

</details>

<details>
<summary><b>🚫 ROOT CAUSE: Active Response silently suppressed by Wazuh's global active-response whitelist</b></summary>
<br>

> Rule `100010` confirmed firing correctly in `alerts.log` on every test, but **zero active-response dispatch ever occurred** — no execution log, no error, total silence, across multiple restart-and-retest cycles. Initially suspected a config typo, a missing `jq` dependency, or an agent/execd connectivity issue — all methodically ruled out one at a time (script manually verified working when piped test JSON directly; `wazuh-execd` confirmed alive and running on the agent; `wazuh-analysisd -t` config validation passed clean).
>
> Root-caused via `grep -i "white" /var/ossec/logs/ossec.log` on the Manager, revealing:
> ```
> White listing IP: '127.0.0.1'
> 2 IPs in the white list for active response.
> ```
> Wazuh's `<global><white_list>` block in `ossec.conf` — which applies globally to **every** active-response command, not just `firewall-drop` — includes `127.0.0.1`. All brute-force test traffic had been generated as a **loopback attack** (`ssh baduser@127.0.0.1` run from Ubuntu-Victim against itself), so `srcip` always matched the whitelist and Wazuh silently suppressed dispatch regardless of which rule or command was involved.
>
> This is structurally the same lesson as Phase 0's "self-scan traffic invisible to Snort" finding, recurring at a completely different layer of the stack (active-response dispatch suppression vs. packet-capture interface visibility) — a good example of how the same underlying category of mistake (testing against yourself instead of a genuine external source) can resurface in unrelated subsystems.
>
> **Fix in progress:** regenerate the brute-force test **cross-host** (Wazuh-Manager → Ubuntu-Victim's real IP `10.0.2.14`), matching the same substitution pattern already used in Phases 0/1/2, combined with Phase 1's documented FIPS/KexAlgorithms fix (`-o KexAlgorithms=diffie-hellman-group14-sha256`) since the Manager's SSH client is FIPS-restricted.

</details>

<details>
<summary><b>💾 Wazuh-Manager memory pressure causing intermittent SSH test failures</b></summary>
<br>

> While attempting the cross-host brute-force test fix above, SSH connections began intermittently timing out mid-loop (first attempt succeeds, subsequent attempts hang and time out) — happening consistently across two separate testing sessions. Initially suspected a firewall self-lockout repeat of the incident documented in Phase 5 (`firewall-drop` auto-blocking the Manager's own IP), but `iptables -L INPUT` on Ubuntu-Victim showed no DROP rule for `10.0.2.12`, ruling that out.
>
> `free -h` on the Wazuh-Manager revealed the real cause: only **253Mi available out of 2.9Gi total RAM, with zero swap configured** — compared to Ubuntu-Victim's healthy 645Mi available plus 2.1Gi swap as a safety buffer. This is a genuine host-level resource constraint (OpenSearch's memory footprint on the Manager, consistent with the OOM-kill pattern first documented in Phase 0) rather than a Wazuh configuration problem — reassuring in one sense, since it suggests the Active Response binding itself is likely correctly configured and simply needs a cleaner test run under less memory pressure.
>
> **Mitigation in progress:** spacing out the brute-force loop (`sleep 2` between attempts) and bounding each SSH attempt with `-o ConnectTimeout=10` to reduce burst load on the Manager and prevent one hung connection from stalling the whole test sequence.

</details>

---

## 📌 Current Status Summary

**Completed and verified — full n8n pipeline operational end-to-end, including real-IP validation:**
- ✅ Infrastructure decision made and documented (self-hosted + Cloudflare Tunnel vs. cloud VPS)
- ✅ Security patching applied to host VM with Snort/Wazuh Agent regression-checked pre/post
- ✅ Node.js 20.x LTS + n8n 2.8.4 installed, memory footprint empirically validated (~200Mi)
- ✅ Cloudflare Tunnel (Quick Tunnel mode) installed and validated end-to-end
- ✅ Webhook trigger node built: POST, random path, Header Auth secret
- ✅ IP enrichment node built and **validated against a real public IP** (`8.8.8.8` → genuine Ashburn, VA / Google LLC geolocation, not just the private-range placeholder test)
- ✅ Message-formatting node (Edit Fields) — combines data across multiple upstream nodes with graceful fallback handling
- ✅ Discord notification node — **live alert with real enrichment data delivered**
- ✅ **Production URL confirmed genuinely backend-listening** — workflow Published, webhook fires correctly with zero n8n editor interaction
- ✅ Custom Active Response script (`n8n-notify-debug.sh`) written, deployed with correct `root:wazuh` / `750` permissions matching Wazuh convention, and manually verified functional
- ✅ Wazuh-Manager `ossec.conf` updated with new `<command>`/`<active-response>` blocks bound to rule `100010`, config validated, Manager restarted clean

**Active blocker, root-caused, fix in progress:**
- 🔴 Active Response dispatch not yet firing end-to-end — root cause identified as Wazuh's global active-response IP whitelist silently suppressing dispatch for loopback-sourced test traffic (`127.0.0.1`). Fix (cross-host test traffic) identified and partially executed; currently contending with a secondary, unrelated Wazuh-Manager memory-pressure issue causing intermittent SSH test interruptions.

**Remaining work:**
- ⏳ Complete one clean, full cross-host Active Response trigger test now that the whitelist root cause is understood
- ⏳ Inspect the real Wazuh stdin JSON payload structure (via the debug script) to write correct `jq` parsing paths
- ⏳ Replace `n8n-notify-debug.sh` with the production `n8n-notify.sh` — real `jq` parsing + `curl` POST to the n8n Production URL
- ⏳ Full live test: real Kali-generated (or cross-host) SSH brute-force → rule `100010` fires → Active Response dispatches → n8n enriches → Discord notifies, with zero manual `curl` triggering anywhere in the chain
- ⏳ Decision + migration to a persistent Cloudflare **Named Tunnel** (stable URL, survives restarts) — deferred, Quick Tunnel sufficient for current build/test phase
- ⏳ Final SOC-style alert reference table and before/after evidence, to be completed once the live end-to-end automated trigger is confirmed working

---

<div align="center">

*Built with 🔐 for learning — Phase 6 in progress.*

</div>
