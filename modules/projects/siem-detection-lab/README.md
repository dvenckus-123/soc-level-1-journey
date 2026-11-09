# SIEM Detection Lab
Hello, welcome to my SIEM detection lab. 

# Overview
This project demonstrates configuration, threat simulation and end-to-end detection of different SOC tools.

# Lab Architecture
- **pfSense + Suricata**: Firewall, IDS & IPS.
- **Wazuh**: Log analysis & detection.
- **Windows Server Domain Controller**: Users, computers, security policies. Sysmon and Wazuh agent.
- **Windows 11 workstation**: Domain-joined workstation. Sysmon and Wazuh agent.
- **Kali Linux**: Attack machine used to simulate threats.
- **VirtualBox**: Virtual infrastructure.

# Lab Topology
- **External zone**: External_WAN. Simulates the public internet. 
- **Internal zone**: Secure_LAN. Simulates an isolated corporate office.

Diagram here

# Telemetry Data Flows
-  **Data Generation(The sources)**:
   * Endpoint & server activity: Windows Event logs & Sysmon logs.
   * Firewall activity: Logs from firewall & IDS/IPS.
-  **Data Collection & Shipping(The transport)**:
   * Lightweight agent: Wazuh agent sends logs to Wazuh SIEM.
   * Network protocols: Pfsense acts as it's own shipper and streams logs over the network to Wazuh SIEM using remote logging function.
-  **Data Ingestion & Analysis(The destination)**:
   * Wazuh: Central location for logs.
