# Build Log

## 1. VirtualBox Network Setup (External_WAN / Secure_LAN)
What the two networks are, why they're separate and how to create them. 

- **External_WAN:** Represents the untrusted internet or an external attacker network.
- **Secure_LAN:** Represents trusted internal corporate network.


pfSense sits in the middle and is the security checkpoint between these two networks. All traffic must pass through pfSense. 

<img width="471" height="368.5" alt="virtual_box_build" src="https://github.com/user-attachments/assets/a0cfcc83-b8c9-49c4-8be6-c08473a4873d" />
