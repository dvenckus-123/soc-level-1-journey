
# Build Log

## 1. VirtualBox Network Setup (External_WAN / Secure_LAN)
What the two networks are, why they're separate and how to create them. 

- **External_WAN:** Represents the untrusted internet or an external attacker network.
- **Secure_LAN:** Represents trusted internal corporate network.


pfSense sits in the middle and is the security checkpoint between these two networks and decides what traffic is allowed between them. 

<img width="707" height="553" alt="topology (1)" src="https://github.com/user-attachments/assets/08659e33-18d1-4e7f-8b7d-39c92fbe187f" />


### VirtualBox Networking Modes
I will use Oracle's documentation on [networking modes](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html#networkingmodes) to configure the lab as follows:
- **NAT Network:** Creates a small simulated internet where Kali and pfSense WAN interface live. VirtualBox assigns them IP addresses. They can communicate with each other and can also access the real internet via VirtualBox networking engine. 
- **Internal Network:** Creates a private virtual switch where the servers and workstations are isolated from the outside world. Devices connected to it can communicate with each other but they rely on pfSense to access the internet.


| Feature | External_WAN | Secure_LAN |
| :--- | :--- | :--- |
| **VirtualBox mode** | NAT Network | Internal Network |
| **Purpose** | Simulated external network | Simulated corporate network |
| **Internet** | Yes | Only through pfSense |
| **Contains** | Kali + pfSense WAN | Windows + Wazuh + pfSense LAN |
| **Security** | Untrusted | Trusted |


