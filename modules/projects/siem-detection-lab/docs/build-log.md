# Build Log

## 1. VirtualBox Network Setup (External_WAN / Secure_LAN)
What the two networks are, why they're separate and how to create them. 

- **External_WAN:** Represents the untrusted internet or an external attacker network.
- **Secure_LAN:** Represents trusted internal corporate network.


pfSense sits in the middle and is the security checkpoint between these two networks and decides what traffic is allowed between them. 

<img width="752" height="602" alt="topology (3)" src="https://github.com/user-attachments/assets/d68d9f31-3f0d-4cc7-bd47-e43ad0ebf8f8" />


### VirtualBox Networking Modes
I will use Oracle's documentation on [networking modes](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html#networkingmodes) to configure the lab as follows:
- **NAT Network:** A shared virtual router that gives VMs internet access using the host's connection. Hosts Kali and the pfSense WAN interface.
- **Internal Network:** A private, isolated virtual switch where VMs can talk to each other, using pfSense for internet access. Hosts the server, workstation, Wazuh, and the pfSense LAN interface.

<br>

|  | External_WAN | Secure_LAN |
| :--- | :--- | :--- |
| **VirtualBox mode** | NAT Network | Internal Network |
| **Purpose** | Simulated external network | Simulated corporate network |
| **Internet Access** | Yes (direct via VirtualBox's NAT engine) | No by default (routed only through pfSense) |
| **VM-to-VM Communications** | Yes (Kali and pfSense WAN can communicate directly) | Yes (Windows, Wazuh, and pfSense LAN can communicate directly) |
| **Host-to-VM Access** | Blocked by default (requires port forwarding rules in NAT Network settings) | Blocked (completely isolated from the host machine) |
| **Contains** | Kali + pfSense WAN | Server + Workstation + Wazuh + pfSense LAN |
| **Security** | Untrusted | Trusted |

<br>
### VM Network Adapter Setup 

| Virtual Machine | Adapter 1 | Adapter 2 |
| :--- | :--- | :--- |
| **pfSense** | NAT Network (`External_WAN`) | Internal Network (`Secure_LAN`) |
| **Kali Linux** | NAT Network (`External_WAN`) | *Disabled* |
| **Windows Server** | Internal Network (`Secure_LAN`) | *Disabled* |
| **Windows 11** | Internal Network (`Secure_LAN`) | *Disabled* |
| **Wazuh SIEM** | Internal Network (`Secure_LAN`) | *Disabled* |


<br>

### IP Addressing
I will use two different private subnets:

|  | Subnet | Device | IP Address |
| :--- | :--- | :--- | :--- |
| **External_WAN** | `172.16.10.0/24` | pfSense WAN | `172.16.10.1` |
| **External_WAN** | `172.16.10.0/24` | Kali Linux | `172.16.10.10` |
| **Secure_LAN** | `192.168.10.0/24` | pfSense LAN | `192.168.10.1` |
| **Secure_LAN** | `192.168.10.0/24` | Windows Server | `192.168.10.10` |
| **Secure_LAN** | `192.168.10.0/24` | Windows 11 | `192.168.10.20` |
| **Secure_LAN** | `192.168.10.0/24` | Wazuh SIEM | `192.168.10.30` |


<br>

### VM Network & IP Configuration
I will use two different private subnets (`172.16.10.0/24` for `External_WAN` and `192.168.10.0/24` for `Secure_LAN`) configured across the virtual machine adapters as follows:

| Virtual Machine | Adapter 1 | Adapter 1 IP | Adapter 2 | Adapter 2 IP |
| :--- | :--- | :--- | :--- | :--- |
| **pfSense** | NAT Network (`External_WAN`) | `172.16.10.1/24` | Internal Network (`Secure_LAN`) | `192.168.10.1/24` |
| **Kali Linux** | NAT Network (`External_WAN`) | `172.16.10.10/24` | *Disabled* | — |
| **Windows Server** | Internal Network (`Secure_LAN`) | `192.168.10.10/24` | *Disabled* | — |
| **Windows 11** | Internal Network (`Secure_LAN`) | `192.168.10.20/24` | *Disabled* | — |
| **Wazuh SIEM** | Internal Network (`Secure_LAN`) | `192.168.10.30/24` | *Disabled* | — |

<br>





### VM Network Adapter Setup

| Virtual Machine | Adapter 1 | Adapter 2 |
| :--- | :--- | :--- |
| **pfSense** | NAT Network (`External_WAN`) | Internal Network (`Secure_LAN`) |
| **Kali Linux** | NAT Network (`External_WAN`) | *Disabled* |
| **Windows Server** | Internal Network (`Secure_LAN`) | *Disabled* |
| **Windows 11** | Internal Network (`Secure_LAN`) | *Disabled* |
| **Wazuh SIEM** | Internal Network (`Secure_LAN`) | *Disabled* |

<br>

### IP Addressing Plan
I will use two private subnets (`172.16.10.0/24` for `External_WAN` and `192.168.10.0/24` for `Secure_LAN`):

| Network | Device / Interface | IP Address | Subnet Mask | Default Gateway | DNS Server |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **External_WAN** | VirtualBox NAT GW | `172.16.10.1` | `255.255.255.0` | — | — |
| **External_WAN** | pfSense WAN | `172.16.10.2` | `255.255.255.0` | `172.16.10.1` | `172.16.10.1` |
| **External_WAN** | Kali Linux | `172.16.10.10` | `255.255.255.0` | `172.16.10.1` | `172.16.10.1` |
| **Secure_LAN** | pfSense LAN | `192.168.10.1` | `255.255.255.0` | *N/A (Router)* | `pfSense DNS` |
| **Secure_LAN** | Windows Server (DC) | `192.168.10.10` | `255.255.255.0` | `192.168.10.1` | `192.168.10.10` (Forwards to `.1`) |
| **Secure_LAN** | Windows 11 | `192.168.10.20` | `255.255.255.0` | `192.168.10.1` | `192.168.10.10` |
| **Secure_LAN** | Wazuh SIEM | `192.168.10.30` | `255.255.255.0` | `192.168.10.1` | `192.168.10.10` |


