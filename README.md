# homelab-siem-wazuh
# SIEM Deployment and Alert Monitoring — Home Lab

## Project Overview
Deployed Wazuh SIEM in a virtualized home lab environment to simulate 
real-world SOC monitoring, alert triage, and incident investigation workflows.

---

## Lab Environment
| Component | Details |
|---|---|
| Host Machine | [Your PC specs — RAM, OS] |
| Virtualization | VirtualBox |
| SIEM Platform | Wazuh 4.7 |
| Attack Machine | Kali Linux |
| Target Machine | Windows 10 |
| Server | Ubuntu Server 22.04 |

---

## Architecture
- Ubuntu Server running Wazuh Manager and Dashboard
- Windows 10 VM connected as Wazuh monitored agent
- Kali Linux VM used to simulate attacks
- All VMs on isolated internal VirtualBox network

---

## Attacks Simulated
| Attack Type | Tool Used | Alert Generated |
|---|---|---|
| Port Scan | Nmap | Network scan detected |
| Brute Force | Hydra | Multiple failed login attempts |
| Privilege Escalation | Manual | Suspicious process execution |

---

## Investigation Findings
[This section gets filled in as you complete the lab]

---

## Incident Report
[Upload your actual incident report here once complete]

---

## Screenshots
[Add screenshots of your Wazuh dashboard here]

---

## Tools Used
- Wazuh SIEM
- Kali Linux
- Nmap
- Hydra
- Ubuntu Server 22.04
- VirtualBox

---

## Key Learnings
- How to deploy and configure a SIEM from scratch
- How to connect endpoints as monitored agents
- How to triage and investigate security alerts
- How to document findings in structured incident reports

---

## References
- Wazuh Documentation: https://documentation.wazuh.com
- TryHackMe SOC Level 1 Path
