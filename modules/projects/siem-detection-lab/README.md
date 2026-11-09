# SIEM Detection Lab
Hello, welcome to my SIEM detection lab. 

# Overview
This project demonstrates configuration, threat simulation and end-to-end detection of different SOC tools.

# Lab Architecture
- **pfSense + Suricata**: Firewall, IDS & IPS.
- **Wazuh**: Log analysis & detection.
- **Windows Server Domain Controller**: Users, computers, security policies.
- **Windows 11 workstation**: Domain-joined workstation with sysmon and Wazuh agent.
- **Kali Linux**: Attack machine used to simulate threats.
- **VirtualBox**: Virtual infrastructure.

# Lab Topology
- **External zone**: External_WAN. Simulates the public internet. 
- **Internal zone**: Secure_LAN. Simulates an isolated corporate office.

Diagram here

# Telemetry Data Flows
- * **Data Generation(The sources)**:
-   *Endpoint Activity*: 
