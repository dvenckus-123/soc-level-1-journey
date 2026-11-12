
# Build Log

## 1. VirtualBox Network Setup
This section explains the virtual infrastructure, how it works and how to set it up. 

- **External_WAN:** Represents the untrusted internet or an external attacker network.
- **Secure_LAN:** Represents trusted internal corporate network.


pfSense sits in the middle and has three roles:
- **Firewall:** Filters traffic between WAN and LAN.
- **Router:** Routes traffic between the two networks.
- **Suricata IDS & IPS:** Monitors network traffic for suspicious activity.

<img width="752" height="677" alt="Untitled Diagram (3)" src="https://github.com/user-attachments/assets/cde3df03-30c5-4c84-9c8e-28532210bf6a" />

<br>

## VirtualBox Networking Modes
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

## VM Network Adapter Setup

| Virtual Machine | Adapter 1 | Adapter 2 |
| :--- | :--- | :--- |
| **pfSense** | NAT Network (`External_WAN`) | Internal Network (`Secure_LAN`) |
| **Kali Linux** | NAT Network (`External_WAN`) | *Disabled* |
| **Windows Server** | Internal Network (`Secure_LAN`) | *Disabled* |
| **Windows 11** | Internal Network (`Secure_LAN`) | *Disabled* |
| **Wazuh SIEM** | Internal Network (`Secure_LAN`) | *Disabled* |

<br>

## IP Addressing Plan

I will use two private RFC1918 IPv4 subnets:

- **External_WAN:** `172.16.10.0/24`
- **Secure_LAN:** `192.168.10.0/24`


### External_WAN — 172.16.10.0/24

| Device / Interface | IP Address | Subnet Mask | Default Gateway | DNS Server |
| :--- | :--- | :--- | :--- | :--- |
| VirtualBox NAT Network Gateway | `172.16.10.1` | `255.255.255.0` | — | — |
| pfSense WAN | `172.16.10.2` | `255.255.255.0` | `172.16.10.1` | `8.8.8.8` (Google) |
| Kali Linux | `172.16.10.10` | `255.255.255.0` | `172.16.10.1` | `8.8.8.8` (Google) |

<br>

**Note 1:** When you create a NAT Network in VirtualBox with an IP range such as `172.16.10.0/24`, VirtualBox assigns the first address `.1` to the NAT Network Gateway. You can confirm this in terminal with `VBoxManage natnetwork list`. 

**Note 2:** Because External_WAN uses a private range, pfSense’s ‘Block private networks’ option on WAN must be disabled, or legitimate traffic from Kali is dropped at the firewall.

### Secure_LAN — 192.168.10.0/24

| Device / Interface | IP Address | Subnet Mask | Default Gateway | DNS Server |
| :--- | :--- | :--- | :--- | :--- |
| pfSense LAN | `192.168.10.1` | `255.255.255.0` | None (gateway interface) | pfSense own DNS Resolver |
| Windows Server (Domain Controller) | `192.168.10.10` | `255.255.255.0` | `192.168.10.1` | `192.168.10.10` |
| Windows 11 | `192.168.10.20` | `255.255.255.0` | `192.168.10.1` | `192.168.10.10` |
| Wazuh SIEM | `192.168.10.30` | `255.255.255.0` | `192.168.10.1` | `192.168.10.10` |

<br>

**Note 3:** The Domain Controller will not use DHCP to hand out IP addresses. We don't want IP addresses to change. Firewalls and server should always have the same static IP address. DNS will be used on the Domain Controller. Secure_LAN devices will point to Domain Controller for DNS resolution and to join the domain. 

## Implementation in VirtualBox GUI

Create External_WAN:

`Network > NAT Networks > Create`
- Name: External_WAN
- IPv4 Prefix: `172.16.10.0/24`
- Enable DHCP: Yes

Create Secure_LAN:

Typed into each VMs network adapter settings rather than created separately.
