# Build Log

## 1. VirtualBox Network Setup (External_WAN / Secure_LAN)
What the two networks are, why they're separate and how to create them. 

- **External_WAN:** Represents the untrusted internet or an external attacker network.
- **Secure_LAN:** Represents trusted internal corporate network.


pfSense sits in the middle and is the security checkpoint between these two networks and decides what traffic is allowed between them. 

<img width="752" height="602" alt="topology (3)" src="https://github.com/user-attachments/assets/d68d9f31-3f0d-4cc7-bd47-e43ad0ebf8f8" />


### VirtualBox Networking Modes
I will use Oracle's documentation on [networking modes](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html#networkingmodes) to configure the lab as follows:
- **NAT Network (`External_WAN`):** A shared virtual network that allows connected VMs to communicate with each other and access the internet through VirtualBox's NAT engine. It hosts Kali Linux and the pfSense WAN interface. This simulates the untrusted external network.
- **Internal Network (`Secure_LAN`):** A private, isolated virtual switch where connected VMs can communicate with each other but cannot directly access the host or external networks. The VMs rely on pfSense for internet access and communication with other networks. It hosts Windows Server, Windows 11, Wazuh, and the pfSense LAN interface.

<br>

| Feature | External_WAN | Secure_LAN |
| :--- | :--- | :--- |
| **VirtualBox mode** | NAT Network | Internal Network |
| **Purpose** | Simulated external network | Simulated corporate network |
| **Internet Access** | Yes (through VirtualBox's NAT engine) | No direct access; routed through pfSense |
| **VM-to-VM Communication** | Yes (Kali and pfSense WAN can communicate) | Yes (Windows, Wazuh, and pfSense LAN can communicate) |
| **Host-to-VM Access** | Incoming access blocked by default; requires NAT Network port forwarding | No direct host access |
| **Contains** | Kali Linux + pfSense WAN | Windows Server + Windows 11 + Wazuh + pfSense LAN |
| **Security Zone** | Untrusted | Trusted internal network |

<br>

### VM Network Adapter Setup

| Virtual Machine | Adapter 1 | Adapter 2 |
| :--- | :--- | :--- |
| **pfSense** | NAT Network (`External_WAN`) | Internal Network (`Secure_LAN`) |
| **Kali Linux** | NAT Network (`External_WAN`) | *Disabled* |
| **Windows Server** | Internal Network (`Secure_LAN`) | *Disabled* |
| **Windows 11** | Internal Network (`Secure_LAN`) | *Disabled* |
| **Wazuh SIEM** | Internal Network (`Secure_LAN`) | *Disabled* |

### IP Addressing Plan

I will use two private IPv4 subnets:

- **External_WAN:** `172.16.10.0/24`
- **Secure_LAN:** `192.168.10.0/24`

#### External_WAN — 172.16.10.0/24

| Device / Interface | IP Address | Subnet Mask | Default Gateway | DNS Server |
| :--- | :--- | :--- | :--- | :--- |
| VirtualBox NAT Network Gateway | `172.16.10.1` | `255.255.255.0` | — | — |
| pfSense WAN | `172.16.10.2` | `255.255.255.0` | `172.16.10.1` | DHCP-provided DNS |
| Kali Linux | `172.16.10.10` | `255.255.255.0` | `172.16.10.1` | DHCP-provided DNS |

#### Secure_LAN — 192.168.10.0/24

| Device / Interface | IP Address | Subnet Mask | Default Gateway | DNS Server |
| :--- | :--- | :--- | :--- | :--- |
| pfSense LAN | `192.168.10.1` | `255.255.255.0` | None (gateway interface) | pfSense DNS Resolver |
| Windows Server (Domain Controller) | `192.168.10.10` | `255.255.255.0` | `192.168.10.1` | `192.168.10.10` |
| Windows 11 | `192.168.10.20` | `255.255.255.0` | `192.168.10.1` | `192.168.10.10` |
| Wazuh SIEM | `192.168.10.30` | `255.255.255.0` | `192.168.10.1` | `192.168.10.10` |
