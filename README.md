# Wazuh SIEM + Shuffle SOAR Detection & Response Pipeline

> **Cybersecurity Homelab · Project 1** | Built by Venkata Nukala

![Wazuh](https://img.shields.io/badge/Wazuh-4.9.0-blue) ![Shuffle](https://img.shields.io/badge/Shuffle-SOAR-blue) ![Status](https://img.shields.io/badge/Pipeline-Working-green) ![Auto-Block](https://img.shields.io/badge/Auto--Block-Active-red)

## Overview

Built a detection and **automated response** pipeline from scratch using Wazuh SIEM and Shuffle SOAR on a local VMware homelab. Simulated real attacker behaviour from a Kali Linux VM — network reconnaissance with Nmap (T1046) and SSH brute-force with Hydra (T1110) — and built a fully automated pipeline that detects, enriches, notifies, and **automatically blocks** the attacker at the firewall — no human action required.

## Architecture

```
[Kali Linux 192.168.68.131]
        |
        |  Nmap scan (T1046) + Hydra SSH brute force (T1110)
        v
[Ubuntu Server 192.168.68.130]  <-- Target
        |
        |  /var/log/auth.log (Docker volume mount)
        v
[Wazuh SIEM - Docker]
        |
        |  Alert Level >= 10 · Rule 5758 fires
        v
[Shuffle SOAR - Webhook]
        |
        |-- VirusTotal v3 (IP enrichment)
        |-- Email Alert (Incident Notification)
        |-- SSH Auto-Block (ufw insert 1 deny from <attacker_ip>)
        v
[Ubuntu Firewall - UFW]
        |
        |  Attacker IP blocked at position 1 — all traffic dropped
        v
[Hydra: ERROR — Timeout connecting to 192.168.68.130]
```

## Lab Environment

| VM | OS | IP | Role | Resources |
|---|---|---|---|---|
| Ubuntu Server | Ubuntu 22.04 LTS | 192.168.68.130 | Wazuh Manager + Shuffle SOAR | 4 vCPU · 8 GB RAM |
| Kali Linux | Kali 2024.x | 192.168.68.131 | Attacker VM | 4 vCPU · 4 GB RAM |
| Windows 10 | Windows 10 | 192.168.68.1 | Wazuh Agent (host) | Host machine |

All VMs run on VMware Workstation Pro on the host machine.

## Attack Simulations

### T1046 — Network Service Scanning (Nmap)
```bash
# Aggressive full-port SYN scan
nmap -sS -A 192.168.68.130
```
**Result:** All ports filtered by firewall — scan logged, below level 10 threshold, no email triggered. Confirms passive detection is working without alert fatigue.

### T1110 — Brute Force (Hydra)
```bash
# SSH brute force with 4 threads
hydra -l mrrobot -P /usr/share/wordlists/rockyou.txt ssh://192.168.68.130 -t 4 -V
```
**Result:** Rule 5758 fired → Shuffle webhook triggered → VirusTotal queried attacker IP → email received within 2 minutes.

---

## Screenshots

### Wazuh Alert — Rule 5758 Fired
![Wazuh Alert Detail](wazuh%20alert.png)

Rule 5758 fired at level 8. Attacker IP `192.168.68.131` mapped to MITRE T1110 Brute Force / Credential Access automatically by Wazuh's built-in rule set.

---

### Wazuh Agents Dashboard — Both VMs Active
![Wazuh Agents Dashboard](Wazuh%20agents.png)

Ubuntu target (`MrRobot`) and Kali attacker both visible and reporting to the Wazuh manager.

---

### Shuffle SOAR — Full Pipeline
![Shuffle Workflow](final%20shuffle%20workflow.png)

Webhook → Change Me (alert parser) → VirusTotal v3 (IP enrichment) → Email → SSH Auto-Block. Full pipeline visible in Build view, running status confirmed.

---

### Shuffle Debug — Execution Overview
![Shuffle Debug Overview](debug1.png)

Execution status: **FINISHED**. Source: webhook. Started 16:19:56, finished 16:20:14. Full Wazuh JSON payload visible flowing through each node.

---

### Shuffle Debug — Wazuh Alert Payload
![Shuffle Debug Payload](debug2.png)

The Change Me node showing the parsed Wazuh alert: severity, title, rule ID, timestamp, and all_fields (10 items). This is the raw JSON Wazuh sent to Shuffle.

---

### Shuffle Debug — VirusTotal + Email Result
![Shuffle Debug Results](debug3.png)

**VirusTotal v3:** `status: 200`, IP `192.168.68.131` queried, `success: true`.  
**Email 1:** `success: true`, `reason: "Email sent to nukalavenkatanarsimahabharghav@gmail.com"`.  
**SSH Auto-Block:** `success: true` — UFW deny rule inserted at position 1 on Ubuntu Server.  
End-to-end confirmation that every node in the pipeline ran successfully.

---

### Email Alert — Inbox Proof
![Email Alert](Email%20Alert.png)

The actual email received in Gmail. Subject: **Wazuh Alert**. Body contains the rule description, agent name, severity level 10, timestamp, and VirusTotal enrichment results — `"malicious": 0, "harmless": 56` — embedded directly in the notification. Full pipeline confirmed end-to-end.

---

## Sample Alert

```json
{
  "rule": {
    "id": "5758",
    "description": "Maximum authentication attempts exceeded",
    "level": 8,
    "mitre": {
      "technique": ["Brute Force"],
      "tactic": ["Credential Access"],
      "id": ["T1110"]
    }
  },
  "agent": { "name": "MrRobot" },
  "data": {
    "srcip": "192.168.68.131",
    "dstuser": "mrrobot"
  }
}
```

---

## Reproduce This Lab

The full Shuffle workflow is exported as `Wazuh Enrichment.json` in this repo.

**To import into your own Shuffle instance:**
1. Open Shuffle → **Workflows** → **Import workflow**
2. Upload `Wazuh Enrichment.json`
3. Update the Email node with your SMTP credentials
4. Update the VirusTotal node with your API key
5. Copy the new webhook URL and paste it into your `wazuh_manager.conf` `<hook_url>` field
6. Restart the Wazuh manager container

You'll have the full Webhook → VirusTotal → Email → SSH Auto-Block pipeline running in under 5 minutes.

**Note for SSH auto-block node:** The SSH node uses key-based authentication. You'll need to:
1. Generate an SSH key pair on your target host: `ssh-keygen -t rsa -b 4096 -f ~/.ssh/shuffle_key -N ""`
2. Add the public key to `~/.ssh/authorized_keys`
3. Upload the private key file to Shuffle → Admin → Files
4. Reference the file ID in the SSH node's "Private key file id" field
5. Add a sudoers rule so the SSH user can run UFW without a password: `username ALL=(ALL) NOPASSWD: /usr/sbin/ufw`

---

## Key Learnings — What Actually Broke

**1. Docker container isolation hides the host filesystem**  
Wazuh runs in a Docker container. Even though `wazuh_manager.conf` pointed to `/var/log/auth.log`, the container couldn't see the host file — Docker doesn't expose the host filesystem by default. Fix: added a read-only bind mount in `docker-compose.yml`:
```yaml
volumes:
  - /var/log/auth.log:/var/log/auth.log:ro
```

**2. XML parser crashes on split closing tags**  
When saving `wazuh_manager.conf` in nano, the `</integration>` closing tag got split across two lines. Wazuh's XML parser threw error 1101 (mismatched tag) and the manager wouldn't start. Fix: opened the file in nano, found the split line, and put it back on one line.

**3. Alert threshold flooding the inbox**  
The default Shuffle trigger level was 3 — meaning every syslog event fires an email. Ran one Hydra scan and got 200+ emails in minutes. Fix: raised the `<level>` in the `<integration>` block in `wazuh_manager.conf` to 10:
```xml
<integration>
  <name>shuffle</name>
  <hook_url>YOUR_WEBHOOK_URL</hook_url>
  <level>10</level>
  <alert_format>json</alert_format>
</integration>
```

**4. Agent/manager version mismatch**  
`apt` installed Wazuh agent 4.14.9 but the manager runs 4.9.0 — they refused to handshake across major versions. Fix: manually downgraded the agent to match the manager version exactly.

**5. SSH authentication in Shuffle**  
The pre-filled private key file ID in the Shuffle SSH node didn't correspond to a valid key. Fix: generated a new RSA 4096-bit key pair on Ubuntu, added the public key to `~/.ssh/authorized_keys`, uploaded the private key to Shuffle Files, and referenced the new file ID.

**6. Attacker IP variable path in Shuffle**  
The Wazuh webhook payload wraps all alert fields under `all_fields` in Shuffle's format. The attacker IP is at `$change_me.all_fields.data.srcip` — not `$change_me.data.srcip` directly. Expanded the Runtime Argument view in Shuffle's debug pane to trace the exact JSON path.

**7. UFW rule ordering matters**  
UFW evaluates rules top-to-bottom. A `ufw deny from <IP>` adds the rule at the bottom — below the existing `allow 22` rule — making it ineffective for SSH traffic. Fix: use `ufw insert 1 deny from <IP>` to place the deny rule at position 1, above all allow rules.

---

## What I'd Do Differently

- Install dedicated Wazuh agents on each VM instead of using Docker volume mounts — cleaner architecture and works out of the box at scale
- Set alert level thresholds **before** running the first attack, not after the inbox floods
- Run `docker compose logs -f` while configuring to catch XML parse errors immediately instead of after a failed restart
- Keep a running doc during the build — the interesting stuff happens in the failures, not the successes
- Set up SSH key-based auth for Shuffle from the start — the extra setup is worth it for automated response nodes

---

## Tech Stack

| Tool | Version | Purpose |
|---|---|---|
| Wazuh | 4.9.0 | SIEM — log ingestion, detection, alerting |
| Shuffle SOAR | Latest | Automation — webhook, enrichment, email |
| OpenSearch | Bundled | Log storage and dashboard |
| Docker Compose | Latest | Container orchestration |
| Kali Linux | 2024.x | Attacker VM — Nmap, Hydra |
| Ubuntu Server | 22.04 LTS | Target VM |
| Nmap | Latest | Network recon (T1046) |
| Hydra | Latest | SSH brute force (T1110) |
| VMware Workstation Pro | Latest | Hypervisor |
| MITRE ATT&CK | — | Threat framework reference |
