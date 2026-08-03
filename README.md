# Building an OT Incident Response Plan for a Simulated Utility Company

**Course:** CY376 – Network Monitoring, Security and Auditing  
**Student:** Nana Kwaku Tweneboah  
**Index Number:** FCM.41.018.250.23  
**Institution:** University of Mines and Technology  
**Team:** Blue Team  
**Date:** August 2026  

---

## 1. Project Overview

This project designs and implements a **Blue Team Operational Technology (OT) Incident Response** capability for a simulated municipal water utility called **AquaGrid Utilities**.

The laboratory demonstrates how to:

- Deploy a production-grade open-source SIEM (**Wazuh**) using Docker Compose on Ubuntu Server
- Instrument an ICS honeypot (**Conpot**) that emulates Modbus and S7comm services
- Create custom detection rules mapped to **MITRE ATT&CK for ICS**
- Simulate both legitimate process traffic and attacker behaviour
- Detect, triage and document early-stage intrusion activity against industrial assets

All work was performed inside an isolated virtual laboratory. No live production systems were targeted.

---

## 2. Laboratory Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────────┐
│  Attacker VM    │────▶│  Virtual Firewall│────▶│  OT / ICS Segment       │
│  (Kali / Ubuntu)│     │                  │     │  - Conpot (Honeypot)    │
└─────────────────┘     └──────────────────┘     │  - Simulated PLC/RTU    │
                                                 │  - Engineering WS       │
                                                 └────────────┬────────────┘
                                                              │
                                                 ┌────────────▼────────────┐
                                                 │  Wazuh Stack            │
                                                 │  (Docker Compose)       │
                                                 │  - Manager              │
                                                 │  - Indexer              │
                                                 │  - Dashboard            │
                                                 │  - Agent(s)             │
                                                 └─────────────────────────┘
```

### Virtual Machines Used

| Role                        | OS              | Purpose                                      |
|----------------------------|-----------------|----------------------------------------------|
| Wazuh Host                 | Ubuntu Server   | Runs the full Wazuh stack via Docker Compose |
| OT / Grid Simulator        | Ubuntu Server   | Runs Conpot honeypot + simulated PLC traffic |
| Attacker / Engineering WS  | Ubuntu / Kali   | Generates legitimate & malicious traffic     |

---

## 3. Prerequisites

- Ubuntu Server 22.04 LTS (or 24.04)
- Minimum 8 GB RAM, 4 vCPU, 40 GB disk for the Wazuh host
- Docker + Docker Compose plugin
- Basic familiarity with the Linux command line

### Install Docker & Docker Compose (Ubuntu Server)

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add current user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker compose version
```

---

## 4. Deploying Wazuh with Docker Compose

### 4.1 Clone the official Wazuh Docker repository

```bash
cd ~
git clone https://github.com/wazuh/wazuh-docker.git -b v4.7.5 --depth=1
cd wazuh-docker/single-node
```

> You can also use the multi-node profile later for a more production-like layout.

### 4.2 Generate certificates (required)

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

### 4.3 Start the Wazuh stack

```bash
docker compose up -d
```

### 4.4 Verify containers are healthy

```bash
docker compose ps
docker compose logs -f wazuh-manager
```

Expected containers:
- `wazuh.manager`
- `wazuh.indexer`
- `wazuh.dashboard`

### 4.5 Access the Dashboard

- URL: `https://<WAZUH-HOST-IP>`
- Default credentials: `admin` / `SecretPassword` (change immediately)

```bash
# Change the default password (recommended)
docker compose exec wazuh-manager /var/ossec/bin/wazuh-passwords-tool.sh -u admin -p 'YourStrongPasswordHere'
```

---

## 5. Deploying the Conpot ICS Honeypot

On the **OT / Grid Simulator** Ubuntu Server:

```bash
# Install dependencies
sudo apt update
sudo apt install -y python3-pip python3-venv git

# Create project directory
mkdir -p ~/ot-lab && cd ~/ot-lab
python3 -m venv conpot-env
source conpot-env/bin/activate

# Install Conpot
pip install conpot

# Copy default template and customise
cp -r $(python -c "import conpot, os; print(os.path.dirname(conpot.__file__))")/templates/default ./conpot-template
```

### 5.1 Basic Conpot configuration (Modbus + S7comm)

Edit `conpot-template/template.xml` (or use the simplified config below).

Create a simple launch script:

```bash
cat > ~/ot-lab/start-conpot.sh << 'EOF'
#!/bin/bash
source ~/ot-lab/conpot-env/bin/activate
conpot -f --template default -l /var/log/conpot/conpot.log --logfile /var/log/conpot/conpot.json
EOF

chmod +x ~/ot-lab/start-conpot.sh
sudo mkdir -p /var/log/conpot
sudo chown $USER:$USER /var/log/conpot
```

### 5.2 Run Conpot

```bash
~/ot-lab/start-conpot.sh
```

Conpot will listen on:
- **Modbus/TCP** → port `502`
- **S7comm** → port `102`

---

## 6. Installing and Enrolling a Wazuh Agent

On the machine that should report events (Engineering Workstation or the Conpot host):

```bash
# Download and install the agent (Ubuntu)
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list

sudo apt update
sudo apt install -y wazuh-agent

# Point the agent to your Wazuh manager
sudo sed -i 's/MANAGER_IP/<WAZUH-MANAGER-IP>/' /var/ossec/etc/ossec.conf

# Start and enable the agent
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### Enrol the agent (on the manager)

```bash
docker compose exec wazuh-manager /var/ossec/bin/manage_agents
# Follow the interactive menu: A (add), then extract the key
```

Or use the one-liner method:

```bash
# On agent
sudo /var/ossec/bin/agent-auth -m <WAZUH-MANAGER-IP>
sudo systemctl restart wazuh-agent
```

---

## 7. Custom Detection Rules (OT / ICS)

Create a local rules file on the Wazuh manager:

```bash
docker compose exec wazuh-manager bash
# Inside the container:
nano /var/ossec/etc/rules/local_rules.xml
```

Paste the following rules:

```xml
<!-- ====================== ICS / Conpot Rules ====================== -->

<!-- Base rule: any Conpot log -->
<group name="ics,conpot,">
  <rule id="100200" level="3">
    <decoded_as>json</decoded_as>
    <field name="program">conpot</field>
    <description>Conpot ICS honeypot event</description>
    <group>ics,conpot,</group>
  </rule>

  <!-- Any interaction with the honeypot -->
  <rule id="100301" level="5">
    <if_sid>100200</if_sid>
    <description>ICS Honeypot interaction detected</description>
    <mitre>
      <id>T1046</id>
    </mitre>
    <group>ics,conpot,reconnaissance,</group>
  </rule>

  <!-- Unauthorised Modbus write (function codes 5,6,15,16) -->
  <rule id="100215" level="12">
    <if_sid>100200</if_sid>
    <match>function_code.: (5|6|15|16)</match>
    <description>ICS: Unauthorised Modbus write request against Conpot</description>
    <mitre>
      <id>T0801</id>
      <id>T0836</id>
    </mitre>
    <group>ics,modbus,conpot,critical,</group>
  </rule>

  <!-- Protocol anomaly -->
  <rule id="100201" level="10">
    <if_sid>100200</if_sid>
    <match>exception|illegal|unknown</match>
    <description>ICS protocol anomaly detected</description>
    <group>ics,modbus,anomaly,</group>
  </rule>
</group>
```

Restart the manager:

```bash
docker compose restart wazuh-manager
```

---

## 8. Forwarding Conpot Logs to Wazuh

On the Conpot host, configure the Wazuh agent to read the JSON log:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add inside the `<ossec_config>` block:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/conpot/conpot.json</location>
</localfile>
```

Restart the agent:

```bash
sudo systemctl restart wazuh-agent
```

---

## 9. Simulating Attacks (for testing)

### 9.1 Simple Modbus scan / write (from attacker machine)

```bash
# Install tools
sudo apt install -y python3-pip
pip3 install pymodbus

# Read holding registers (legitimate-looking)
python3 -c "
from pymodbus.client import ModbusTcpClient
client = ModbusTcpClient('<CONPOT-IP>', port=502)
client.connect()
result = client.read_holding_registers(0, 10, slave=1)
print(result.registers if not result.isError() else result)
client.close()
"

# Unauthorised write (should trigger rule 100215)
python3 -c "
from pymodbus.client import ModbusTcpClient
client = ModbusTcpClient('<CONPOT-IP>', port=502)
client.connect()
result = client.write_register(0, 999, slave=1)
print(result)
client.close()
"
```

### 9.2 Port scan

```bash
nmap -sS -p 102,502,44818 <CONPOT-IP>
```

### 9.3 SSH brute-force style test (optional)

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://<CONPOT-IP>
```

---

## 10. Useful Wazuh Commands

```bash
# Check agent status
docker compose exec wazuh-manager /var/ossec/bin/agent_control -l

# View manager logs
docker compose logs -f wazuh-manager

# Restart whole stack
docker compose restart

# Stop stack
docker compose down

# Full reset (careful – deletes data)
docker compose down -v
```

---

## 11. Project Folder Structure (recommended for GitHub)

```
aquagrid-ot-ir/
├── README.md
├── docs/
│   ├── report.pdf
│   └── presentation.pptx
├── docker/
│   └── (optional custom compose overrides)
├── configs/
│   ├── local_rules.xml
│   ├── ossec.conf.snippets
│   └── conpot-template/
├── scripts/
│   ├── start-conpot.sh
│   ├── attack-modbus-write.py
│   └── attack-scan.sh
├── evidence/
│   ├── screenshots/
│   └── sample-alerts/
└── .gitignore
```

### Suggested `.gitignore`

```
*.log
*.json
.env
__pycache__/
conpot-env/
*.pcap
.DS_Store
```

---

## 12. Troubleshooting Notes (from the lab)

| Problem                              | Solution                                                                 |
|--------------------------------------|--------------------------------------------------------------------------|
| Agent shows “Never connected”        | Re-run `agent-auth` and restart agent                                    |
| Certificate mismatch after snapshot  | Remove agent on manager + delete `client.keys` on agent + re-enrol       |
| Conpot logs not appearing            | Check file permissions and `<localfile>` path in `ossec.conf`            |
| Dashboard not reachable              | Confirm ports 443 / 5601 are open and containers are healthy             |
| High false positives on FIM          | Update baseline after intentional config changes                         |

---

## 13. Mapping to MITRE ATT&CK for ICS

| Rule ID  | Technique                          | ATT&CK ID |
|----------|------------------------------------|-----------|
| 100301   | Network Service Scanning           | T1046     |
| 100215   | Exploit Public-Facing Application  | T0801     |
| 100215   | Modify Parameter                   | T0836     |
| 100201   | Protocol anomaly / Impair Process  | T0836     |

---

## 14. How to Reproduce This Lab

1. Provision three Ubuntu Server VMs (or use one powerful host with multiple containers).
2. Install Docker on the Wazuh host and bring up the stack (Section 4).
3. Deploy Conpot on the OT simulator host (Section 5).
4. Install and enrol Wazuh agents (Section 6).
5. Add the custom rules (Section 7).
6. Configure log forwarding (Section 8).
7. Generate test traffic (Section 9).
8. Observe alerts in the Wazuh Dashboard and export evidence for the report.

---

## 15. License & Academic Integrity

This repository contains laboratory configurations, detection rules and documentation produced for the CY376 end-of-semester project at the University of Mines and Technology.  

All attack simulations were performed exclusively inside an isolated, instructor-approved laboratory environment.  

**Do not** use any of the attack scripts against systems you do not own or do not have explicit written permission to test.

---

**Author:** Nana Kwaku Tweneboah  
**Index:** FCM.41.018.250.23  
**Course:** CY376 – Network Monitoring, Security and 
