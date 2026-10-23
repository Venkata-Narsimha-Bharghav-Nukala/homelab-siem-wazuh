# Wazuh SIEM + Shuffle SOAR Detection & Response Pipeline

> **Cybersecurity Homelab · Project 1** | Built by Venkata Nukala

![Wazuh](https://img.shields.io/badge/Wazuh-4.9.0-blue) ![Shuffle](https://img.shields.io/badge/Shuffle-SOAR-blue) ![Status](https://img.shields.io/badge/Pipeline-Working-green)

## Overview

Built a detection and automated response pipeline from scratch using Wazuh SIEM and Shuffle SOAR on a local VMware homelab. Simulated real attacker behaviour from a Kali Linux VM — network reconnaissance with Nmap (T1046) and SSH brute-force with Hydra (T1110) — and tuned the pipeline to detect, alert, and send automated email notifications for confirmed threats.

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
        |  Automated email dispatch
        v
[Email Alert - Incident Notification]
```

## Lab Environment

| VM | OS | IP | Role | Resources |
|---|---|---|---|---|
| Ubuntu Server | Ubuntu 22.04 LTS | 192.168.68.130 | Wazuh Manager + Shuffle SOAR | 4 vCPU · 8 GB RAM |
| Kali Linux | Kali 2024.x | 192.168.68.131 | Attacker VM | 4 vCPU · 4 GB RAM |
| Windows 10 | Windows 10 | 192.168.68.1 | Wazuh Agent (host) | Host machine |

## Attack Simulations

### T1046 — Network Service Scanning (Nmap)
```bash
# Aggressive full-port SYN scan
nmap -sS -A 192.168.68.130
```
**Result:** All ports filtered by firewall — scan logged, below level 10 threshold, no email triggered.

### T1110 — Brute Force (Hydra)
```bash
# SSH brute force with 4 threads
hydra -l mrrobot -P /usr/share/wordlists/rockyou.txt ssh://192.168.68.130 -t 4 -V
```
**Result:** Rule 5758 fired → Shuffle webhook triggered → email received within 2 minutes.

## Sample Alert

| Field | Value |
|---|---|
| Rule ID | 5758 |
| Description | Maximum authentication attempts exceeded |
| Level | 8 |
| Attacker IP | 192.168.68.131 |
| Target User | mrrobot |
| Log Source | /var/log/auth.log |
| MITRE ID | T1110 |
| Tactic | Credential Access |
| Technique | Brute Force |

## Screenshots

### Wazuh Alert — Rule 5758 Fired
![Wazuh Alert Detail](wazuh%20alert.png)

Rule 5758 fired at level 8. Attacker IP `192.168.68.131` mapped to MITRE T1110 automatically by Wazuh's built-in rule set.

---

### Wazuh Agents Dashboard — Both VMs Active
![Wazuh Agents Dashboard](Wazuh%20agents.png)

Ubuntu target (`MrRobot`) and Kali attacker both visible and reporting to the Wazuh manager.

---

### Shuffle SOAR — Full Pipeline
![Shuffle Workflow](final%20shuffle%20workflow.png)

Webhook → Change Me (alert parser) → VirusTotal v3 (IP enrichment) → Email. Running status confirmed top right.

---

### Shuffle Debug — Execution Proof
![Shuffle Debug Overview](debug1.png)

Status: **FINISHED**. Source: webhook. Started 16:19:56, finished 16:20:14. Full Wazuh JSON payload flowing through each node.

![Shuffle Debug Payload](debug2.png)

Change Me node output — parsed alert showing severity, rule ID, timestamp, and all 10 fields from the Wazuh alert.

![Shuffle Debug Results](debug3.png)

**VirusTotal:** `status: 200`, attacker IP queried successfully. **Email:** `success: true`, delivery confirmed.

---

## Reproduce This Lab

The full Shuffle workflow is exported as `Wazuh Enrichment.json` in this repo.

**To import into your own Shuffle instance:**
1. Open Shuffle → **Workflows** → **Import workflow**
2. Upload `Wazuh Enrichment.json`
3. Update the Email node with your SMTP credentials
4. Update the VirusTotal node with your API key
5. Copy the new webhook URL into your `wazuh_manager.conf` `<hook_url>` field
6. Restart the Wazuh manager container

Full pipeline running in under 5 minutes.

## Key Learnings

1. **Docker container isolation** — Wazuh manager can't see the host filesystem by default. Had to add a read-only volume mount in docker-compose.yml to expose auth.log inside the container.
2. **Version pinning matters** — apt installed Wazuh agent 4.14.9 but the manager runs 4.9.0. They refused to communicate until the agent was manually downgraded.
3. **Alert noise is a real problem** — Default threshold (level 3) sent 200+ emails in minutes. Tuned to level 10. Alert fatigue is a genuine SOC challenge.
4. **MITRE ATT&CK mapping is automatic** — Wazuh's built-in rules map to MITRE techniques automatically. T1110 Brute Force fired on a real Hydra attack.

## What I'd Do Differently

1. **Install dedicated Wazuh agents on each VM from the start** instead of relying on volume mounts — cleaner architecture and works at scale.
2. **Set alert level thresholds before running attacks.** Running Hydra with the default level-3 threshold flooded the inbox and required retroactive tuning.
3. **Document while building** — several troubleshooting steps would have been cleaner with running notes.

## Tech Stack

`Wazuh 4.9.0` `Shuffle SOAR` `OpenSearch` `Docker Compose` `Kali Linux` `Ubuntu 22.04` `Nmap` `Hydra` `VMware Workstation Pro` `MITRE ATT&CK`
