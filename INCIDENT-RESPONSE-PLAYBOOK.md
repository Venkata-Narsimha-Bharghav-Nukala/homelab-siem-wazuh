# Incident Response Playbook — SSH Brute Force Attack

> **Standard:** NIST SP 800-61 Rev. 2 — Computer Security Incident Handling Guide  
> **Incident Type:** Credential Access — Brute Force (MITRE ATT&CK T1110)  
> **Severity:** Medium  
> **Status:** Resolved  
> **Author:** Venkata Nukala  
> **Date:** 2026-10-23

---

## Incident Summary

| Field | Value |
|---|---|
| Incident ID | IR-2026-001 |
| Detection Time | 2026-10-23 16:19:56 UTC |
| Containment Time | 2026-10-23 16:20:14 UTC |
| Total Response Time | ~18 seconds (automated) |
| Attack Type | SSH Brute Force |
| MITRE Technique | T1110 — Brute Force / Credential Access |
| Attacker IP | 192.168.68.131 (Kali Linux VM) |
| Target IP | 192.168.68.130 (Ubuntu Server — MrRobot) |
| Target Account | mrrobot |
| Detection Rule | Wazuh Rule 5758 — Maximum authentication attempts exceeded |
| Alert Level | 8 |
| Outcome | Detected and alerted. No successful authentication. |

---

## Phase 1 — Preparation

### Security Controls in Place

| Control | Tool | Configuration |
|---|---|---|
| Log collection | Wazuh Agent | `/var/log/auth.log` bind-mounted read-only into Wazuh Docker container |
| SIEM alerting | Wazuh 4.9.0 | Rule 5758 enabled; alert level threshold set to 10 for email dispatch |
| Automated response | Shuffle SOAR | Webhook trigger on Wazuh alert → VirusTotal enrichment → email notification |
| IP reputation | VirusTotal v3 | API integrated into Shuffle workflow for attacker IP lookup |
| Notification | Gmail SMTP | Automated email to SOC inbox on confirmed alert |

### Monitoring Scope

- **In scope:** SSH authentication events on Ubuntu Server 192.168.68.130
- **Log source:** `/var/log/auth.log` — parsed by Wazuh agent running on the host
- **Alert threshold:** Level 8+ triggers Wazuh rule; level 10+ triggers Shuffle webhook and email

### Known Environment

```
[Kali Linux 192.168.68.131]  ← Designated attacker VM
[Ubuntu Server 192.168.68.130]  ← Target / Wazuh-monitored endpoint
[Wazuh Manager - Docker]  ← SIEM running on Ubuntu Server
[Shuffle SOAR - Docker]  ← Automation running on Ubuntu Server
```

---

## Phase 2 — Detection & Analysis

### 2.1 Initial Detection

At **16:19:56 UTC**, Wazuh fired **Rule 5758** against the Ubuntu Server agent (`MrRobot`):

```
Rule ID:      5758
Description:  Maximum authentication attempts exceeded
Level:        8
Agent:        MrRobot (192.168.68.130)
Source IP:    192.168.68.131
Target User:  mrrobot
Log Source:   /var/log/auth.log
MITRE ID:     T1110
Tactic:       Credential Access
Technique:    Brute Force
```

### 2.2 Attack Reconstruction

The attacker (`192.168.68.131`) ran Hydra against the SSH service on the target:

```bash
hydra -l mrrobot -P /usr/share/wordlists/rockyou.txt ssh://192.168.68.130 -t 4 -V
```

- **Wordlist:** `rockyou.txt` (~14 million passwords)
- **Threads:** 4 concurrent SSH connections
- **Target account:** `mrrobot`
- **Pattern:** Rapid sequential failed authentication attempts — matched Rule 5758 signature

### 2.3 Automated Enrichment (Shuffle SOAR)

Shuffle received the Wazuh webhook at **16:19:56** and completed enrichment by **16:20:14** (18 seconds total):

**Node 1 — Webhook:** Received full Wazuh alert JSON payload  
**Node 2 — Change Me (Parser):** Extracted severity, rule ID, source IP, timestamp from payload  
**Node 3 — VirusTotal v3:** Queried attacker IP `192.168.68.131`

```json
{
  "malicious": 0,
  "suspicious": 0,
  "undetected": 36,
  "harmless": 56,
  "timeout": 0
}
```

**Node 4 — Email:** Notification dispatched to SOC inbox  
`success: true — Email sent to nukalavenkatanarsimahabharghav@gmail.com`

### 2.4 Threat Assessment

| Factor | Assessment |
|---|---|
| Attacker IP reputation | Clean (VirusTotal: 0 malicious, 56 harmless) — internal lab IP as expected |
| Authentication success | No successful logins detected in auth.log |
| Lateral movement | No — attack contained to SSH on single host |
| Data exfiltration | No evidence |
| Persistence | No — no new accounts, cron jobs, or SSH keys observed |
| Severity | **Medium** — active brute force, no breach |

---

## Phase 3 — Containment, Eradication & Recovery

### 3.1 Immediate Containment

| Action | Method | Status |
|---|---|---|
| Block attacker IP at firewall | `ufw deny from 192.168.68.131` | ✅ Recommended |
| Kill active SSH sessions from attacker | `ss -tp | grep 192.168.68.131` → `kill <pid>` | ✅ Recommended |
| Temporarily disable SSH password auth | Set `PasswordAuthentication no` in `/etc/ssh/sshd_config` | ✅ Recommended |

```bash
# Block attacker IP
sudo ufw deny from 192.168.68.131 to any

# Verify no active sessions
ss -tp | grep 192.168.68.131

# Restart SSH after config change
sudo systemctl restart sshd
```

### 3.2 Eradication

Verify no successful authentication occurred during the attack window:

```bash
# Check for successful logins from attacker IP in the attack window
grep "Accepted" /var/log/auth.log | grep "192.168.68.131"

# Check for new user accounts created
grep "useradd\|adduser" /var/log/auth.log

# Check for new SSH authorized keys
cat /home/mrrobot/.ssh/authorized_keys
```

**Expected result:** No output — confirms no breach occurred.

### 3.3 Recovery

Since no successful authentication occurred, no system recovery is required.

If a breach had occurred:
1. Isolate host from network immediately
2. Preserve disk image for forensic analysis
3. Rotate all credentials on the affected host
4. Audit all accounts for unauthorized changes
5. Restore from last known-good snapshot if compromise is confirmed

### 3.4 Hardening Recommendations

| Recommendation | Priority | Implementation |
|---|---|---|
| Disable SSH password auth; use key-based auth only | High | `PasswordAuthentication no` in `/etc/ssh/sshd_config` |
| Install and configure `fail2ban` | High | Auto-ban IPs after 5 failed attempts |
| Change SSH from default port 22 | Medium | `Port 2222` in `/etc/ssh/sshd_config` |
| Restrict SSH access by IP allowlist | Medium | `ufw allow from <trusted_ip> to any port 22` |
| Enable MFA for SSH | Medium | `libpam-google-authenticator` |

---

## Phase 4 — Post-Incident Activity

### 4.1 Timeline of Events

| Time (UTC) | Event |
|---|---|
| 16:19:56 | Hydra brute force begins from 192.168.68.131 |
| 16:19:56 | Wazuh Rule 5758 fires — alert level 8 |
| 16:19:56 | Shuffle webhook triggered |
| 16:20:10 | VirusTotal query returns — 0 malicious |
| 16:20:14 | Email alert delivered to SOC inbox |
| 16:20:14 | Incident detected, analysed, and notified — **total automated response: 18 seconds** |

### 4.2 Lessons Learned

**What worked well:**
- Wazuh Rule 5758 fired immediately on brute force pattern — no manual review needed
- MITRE T1110 mapping was automatic — no analyst effort required for classification
- Shuffle pipeline completed enrichment and notification in 18 seconds
- VirusTotal enrichment gave instant IP reputation context alongside the alert

**What needs improvement:**

| Gap | Impact | Fix |
|---|---|---|
| No automated blocking — alert only | Attacker could continue indefinitely | Add a Shuffle node to run `ufw deny` via SSH on trigger |
| Alert threshold tuned too low initially (level 3) | 200+ emails from one scan | Always set threshold before running attacks |
| No dedicated Wazuh agent on Kali attacker VM | Attacker-side visibility missing | Install agents on all VMs from the start |
| Email body is unformatted plain text | Hard to parse quickly | Improve Shuffle email template with structured HTML |

### 4.3 Metrics

| Metric | Value |
|---|---|
| Mean Time to Detect (MTTD) | < 1 second (automated rule) |
| Mean Time to Respond (MTTR) | 18 seconds (automated enrichment + notification) |
| False Positive Rate | 0% — attack was real and correctly classified |
| Alerts generated | 1 (after threshold tuning to level 10) |

---

## Appendix — Evidence

| Artifact | Description | Location |
|---|---|---|
| `wazuh alert.png` | Wazuh dashboard — Rule 5758 fired, level 8, MITRE T1110 | GitHub repo |
| `debug1.png` | Shuffle execution overview — FINISHED status, 18s runtime | GitHub repo |
| `debug2.png` | Change Me node — parsed Wazuh alert JSON payload | GitHub repo |
| `debug3.png` | VirusTotal result (status 200) + Email result (success: true) | GitHub repo |
| `Email Alert.png` | Gmail inbox — alert received with enrichment data | GitHub repo |
| `Wazuh Enrichment.json` | Full Shuffle workflow export — importable into any Shuffle instance | GitHub repo |

---

*Playbook written in accordance with NIST SP 800-61 Rev. 2 — Computer Security Incident Handling Guide.*  
*Framework reference: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf*
