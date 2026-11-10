
# Build Log

## 1. VirtualBox Network Setup (External_WAN / Secure_LAN)
What the two networks are, why they're separate and how to create them. 

- **External_WAN:** Represents the untrusted internet or an external attacker network.
- **Secure_LAN:** Represents trusted internal corporate network.


pfSense sits in the middle and is the security checkpoint between these two networks. All traffic must pass through pfSense. 

<img width="707" height="553" alt="topology (1)" src="https://github.com/user-attachments/assets/08659e33-18d1-4e7f-8b7d-39c92fbe187f" />

(https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html#networkingmodes) 

### VirtualBox Networking Modes


| Feature | External_WAN | Secure_LAN |
| :--- | :--- | :--- |
| **VirtualBox mode** | NAT Network | Internal Network |
| **Purpose** | Simulated external network | Simulated corporate network |
| **Internet** | Yes | Only through pfSense |
| **Contains** | Kali + pfSense WAN | Windows + Wazuh + pfSense LAN |
| **Security** | Untrusted | Trusted |


- Placing Kali Linux and pfSense WAN interface inside External_WAN allows access to the internet

  
