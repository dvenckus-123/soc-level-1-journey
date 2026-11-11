# Build Log

## 1. VirtualBox Network Setup (External_WAN / Secure_LAN)
What the two networks are, why they're separate and how to create them. 

- **External_WAN:** Represents the untrusted internet or an external attacker network.
- **Secure_LAN:** Represents trusted internal corporate network.


pfSense sits in the middle and is the security checkpoint between these two networks and decides what traffic is allowed between them. 

<img width="752" height="602" alt="topology (3)" src="https://github.com/user-attachments/assets/d68d9f31-3f0d-4cc7-bd47-e43ad0ebf8f8" />


### VirtualBox Networking Modes
I will use Oracle's documentation on [networking modes](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html#networkingmodes) to configure the lab as follows:
- **NAT Network:** Creates a network where multiple VMs can share a NAT router. This is where Kali and pfSense WAN interface live. They can communicate with each other and can also access the real internet through VirtualBox's networking engine. 
- **Internal Network:** Creates an isolated virtual switch. Only VMs that are explicitly configured to join the exact same named internal network can see and talk to each other. Devices connected to it can communicate with each other but they rely on pfSense to access the internet.

<br>

| Feature | External_WAN | Secure_LAN |
| :--- | :--- | :--- |
| **VirtualBox mode** | NAT Network | Internal Network |
| **How it works** | VirtualBox creates a virtual router and DHCP server shared by VMs on this network, translating their private IPs to the host's IP. | VirtualBox creates an isolated, software-based Ethernet switch linking only VMs assigned to the exact same internal network name. |
| **Purpose** | Simulated external network | Simulated corporate network |
| **Internet / External Access** | Yes (direct via VirtualBox's NAT engine) | No by default (routed only through pfSense) |
| **VM-to-VM Communications** | Yes (Kali and pfSense WAN can communicate directly) | Yes (Windows, Wazuh, and pfSense LAN can communicate directly) |
| **Host-to-VM Access** | Blocked by default (requires port forwarding rules in NAT Network settings) | Blocked (completely isolated from the host machine) |
| **Contains** | Kali + pfSense WAN | Windows + Wazuh + pfSense LAN |
| **Security** | Untrusted | Trusted |


<br>


|  | External_WAN | Secure_LAN |
| :--- | :--- | :--- |
| **VirtualBox mode** | NAT Network | Internal Network |
| **Purpose** | Simulated external network | Simulated corporate network |
| **Internet** | Yes | Only through pfSense |
| **Contains** | Kali + pfSense WAN | Windows + Wazuh + pfSense LAN |
| **Security** | Untrusted | Trusted |


