
# Build Log

## 1. VirtualBox Network Setup (External_WAN / Secure_LAN)
What the two networks are, why they're separate and how to create them. 

- **External_WAN:** Represents the untrusted internet or an external attacker network.
- **Secure_LAN:** Represents trusted internal corporate network.


pfSense sits in the middle and is the security checkpoint between these two networks. All traffic must pass through pfSense. 

<img width="942" height="737" alt="topology" src="https://github.com/user-attachments/assets/668fb586-ce23-4211-8479-01fc85c15e4d" />
