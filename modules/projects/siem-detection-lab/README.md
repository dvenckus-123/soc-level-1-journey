# SIEM Detection Lab
Hello, welcome to my SIEM detection lab. 

# Overview
This project demonstrates configuration, threat simulation and end-to-end detection of different SOC tools.

# Lab Architecture
- **pfSense + Suricata**: Firewall, IDS & IPS.
- **Wazuh**: Log analysis & correlation.
- **Windows Server Domain Controller**: Users, computers, security policies. Sysmon and Wazuh agent.
- **Windows 11 workstation**: Domain-joined workstation. Sysmon and Wazuh agent.
- **Kali Linux**: Attack machine used to simulate threats.
- **VirtualBox**: Virtual infrastructure.

# Lab Topology
- **External zone**: External_WAN. Simulates the public internet. 
- **Internal zone**: Secure_LAN. Simulates an isolated corporate office.

Diagram here

# Telemetry Data Flow Matrix
To effectively monitor and analyse security events, the detection lab uses a centralized logging pipeline. The matrix below outlines how telemetry moves from generation to ingestion:



| **Telemetry Type** | **Data Generation (The Sources)** | **Collection & Shipping (The Transport)** | **Ingestion & Analysis (The Destination)** | **What You Detect / Monitor** |
| :--- | :--- | :--- | :--- | :--- |
| **Endpoint & Server Activity** | Windows Event Logs & Sysmon | **Wazuh Agent** (Lightweight agent over secure port 1514) | **Wazuh SIEM** (Central location) | Process creations, malicious PowerShell commands, registry changes, and local account logons. |
| **Network Activity** | pfSense Firewall Logs | **Remote Logging Protocol** (Syslog streaming natively over the network) | **Wazuh SIEM** (Central location) | Blocked traffic, inbound port scans, and unauthorized access attempts at the perimeter. |
| **Intrusion Detection (IDS/IPS)** | Suricata Logs (Hosted on pfSense) | **Remote Logging Protocol** (Forwarded seamlessly alongside pfSense system logs) | **Wazuh SIEM** (Central location) | Known malware signatures, Command and Control (C2) traffic, and network exploitation attempts. |


# Tech Stack
| Component | Role | Version |
| --- | --- | --- |
| --- | --- | --- |
| --- | --- | --- |
| --- | --- | --- |
| --- | --- | --- |
| --- | --- | --- |
| --- | --- | --- |
| --- | --- | --- |

# Defensive Controls
[link to defensve controls](/projects/siem-detection-lab)

**Summary**:
- File Integrity Monitoring.
- VirusTotal integration.
- Firewall.
- IDS/IPS.


# Simulation: Hydra Brute Force
[link to simulation](/projects/siem-detection-lab)

**Summary**:
- Command & target.

# Detection Results
[link to simulation](/projects/siem-detection-lab)

**Summary**:
- Screenshots/investigations.


# Lessons Learned
What broke, false positives, what I would change.
