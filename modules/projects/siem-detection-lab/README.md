# SIEM Detection Lab
Hello, welcome to my SIEM detection lab. 

## Overview
This project demonstrates configuration, threat simulation and detection of different SOC tools.

## Lab Architecture
- **pfSense + Suricata**: Firewall, IDS & IPS.
- **Wazuh**: Log analysis & correlation.
- **Windows Server Domain Controller**: Users, computers, security policies. Sysmon and Wazuh agent.
- **Windows 11 workstation**: Domain-joined workstation. Sysmon and Wazuh agent.
- **Kali Linux**: Attack machine used to simulate threats.
- **VirtualBox**: Virtual infrastructure.

## Lab Topology
To simulate an internet-based threat actor, traffic is split into an an external zone and an internal corporate zone using two virtual switches:
- **External zone**: External_WAN. Simulates the public internet. 
- **Internal zone**: Secure_LAN. Simulates an isolated corporate office.

Diagram here

## Telemetry Data Flow Matrix
To effectively monitor and analyse security events, the detection lab uses a centralized logging pipeline. The matrix below outlines how telemetry moves from generation to ingestion:



| **Telemetry Type** | **Data Generation (The Sources)** | **Collection & Shipping (The Transport)** | **Ingestion & Analysis (The Destination)** | **What You Detect / Monitor** |
| :--- | :--- | :--- | :--- | :--- |
| **Endpoint & Server Activity** | Windows Event Logs & Sysmon | **Wazuh Agent** (Lightweight agent over secure port 1514) | **Wazuh SIEM** (Central location) | Process creations, malicious PowerShell commands, registry changes, and local account logons. |
| **Network Activity** | pfSense Firewall Logs | **Remote Logging Protocol** (Syslog streaming natively over the network) | **Wazuh SIEM** (Central location) | Blocked traffic, inbound port scans, and unauthorized access attempts at the perimeter. |
| **Intrusion Detection (IDS/IPS)** | Suricata Logs (Hosted on pfSense) | **Remote Logging Protocol** (Forwarded seamlessly alongside pfSense system logs) | **Wazuh SIEM** (Central location) | Known malware signatures, Command and Control (C2) traffic, and network exploitation attempts. |


## Tech Stack

| Component | Role | Version |
| --- | --- | --- |
| pfSense | Perimeter firewall | 2.7.x |
| Suricata | IDS/IPS | 7.x |
| Wazuh | SIEM | 4.x |
| Windows Server 2022 | Domain controller | — |
| Windows 11 Pro | Endpoint | — |
| Kali Linux | Attacker | 2024.x |


## Build Log
Step-by-step setup for every component, in build order, referencing the official documentation throughout. <br> <br>
See [Build Log](docs/build-log.md) for the full walkthrough.


## Defensive Controls
See [Defensive Controls](docs/defensive-controls.md) for more details.

**Network Controls**

* Network segmentation (External_WAN / Secure_LAN)
* Perimeter firewall (pfSense)
* Intrusion detection (Suricata)

**Endpoint Controls**

* Endpoint visibility (Sysmon)
* File Integrity Monitoring (FIM)
* Account lockout policy (Active Directory)

**Detection & Response**

* SIEM / log correlation (Wazuh) (research correlation in wazuh)
* Threat intelligence enrichment (VirusTotal)


## Simulation: Hydra Brute Force
See [Simulation: Hydra Brute Force](docs/simulation-hydra.md) for more details. 

**Summary**:
- Command & target.

## Detection Results
See [Detection Results](docs/detection-results.md) for more details. 

**Summary**:
- Screenshots/investigations.


## Lessons Learned
See [Lessons Learned](docs/lessons-learned.md) for more details.

**Summary**:
- What broke, false positives, what I would change.


Suricata logs:

• Suricata wasn't sending logs out: Suricata keeps alerts local on pfSense by default. Fix: We turned on Send Alerts to System Log in pfSense so it forwarded them over UDP 514.
• Wazuh didn't recognize the format: Wazuh expects Suricata logs in JSON format, not plain text. Fix: We wrote a custom decoder and rule in Wazuh to read pfSense's plain-text alerts.
• Wazuh rejected the regex syntax: Wazuh's default regex engine is very basic and threw an error on our pattern. Fix: We added type="pcre2" to enable standard regex support.
• Missing hostname broke the match: pfSense sent the log without a hostname, tricking Wazuh into thinking suricata was the server name instead of the program name. Fix: We updated the decoder to look for suricata[PID]: anywhere in the text instead of relying on the header.


issue with wazuh logs appearing delayed or missing. 


What happened

• Old logs looked new: After pausing your VMs overnight, Wazuh was still showing last night's logs (23:54) because live events hadn't resumed yet.
• Wrong display timezone: Once live events started coming in, your browser was showing them 8 hours behind (US Pacific Time instead of UTC).
• Normal apps get filtered out: Wazuh hides normal Sysmon activity (like opening Notepad) and only alerts on suspicious commands.

How we fixed it

• Restarted the services: Restarted the Wazuh services on your server and Windows 11 PC so live logs started flowing again.
• Set Dashboard time to UTC: Changed the Wazuh Dashboard timezone setting to UTC so the timestamps matched Windows Event Viewer.
• Tested with a suspicious command: Ran an encoded PowerShell command to prove Sysmon alerts were working and showing up on time.

