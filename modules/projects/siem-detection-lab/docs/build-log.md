
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

> **Note 1:** When you create a NAT Network in VirtualBox with an IP range such as `172.16.10.0/24`, VirtualBox assigns the first address `.1` to the NAT Network Gateway. You can confirm this in terminal with `VBoxManage natnetwork list`. 

> **Note 2:** Because External_WAN uses a private range, pfSense’s ‘Block private networks’ option on WAN must be disabled, or legitimate traffic from Kali is dropped at the firewall.

### Secure_LAN — 192.168.10.0/24

| Device / Interface | IP Address | Subnet Mask | Default Gateway | DNS Server |
| :--- | :--- | :--- | :--- | :--- |
| pfSense LAN | `192.168.10.1` | `255.255.255.0` | None (gateway interface) | pfSense DNS Resolver (for own lookups only) |
| Windows Server (Domain Controller) | `192.168.10.10` | `255.255.255.0` | `192.168.10.1` | `192.168.10.10` |
| Windows 11 | `192.168.10.20` | `255.255.255.0` | `192.168.10.1` | `192.168.10.10` |
| Wazuh SIEM | `192.168.10.30` | `255.255.255.0` | `192.168.10.1` | `192.168.10.10` |

<br>

> **Note 3:** The Domain Controller will not use DHCP to hand out IP addresses. We don't want IP addresses to change. Firewalls and servers should always have the same static IP address. 

> **Note 4:** DNS will be used on the Domain Controller. Secure_LAN devices will point to DC for internal DNS queries and to join the domain. The DC forwards external DNS queries to pfSense because Secure_LAN has no other route to the internet.

## Implementation in VirtualBox GUI

Create External_WAN:

Network > NAT Networks > Create
- **Name:** `External_WAN`
- **IPv4 Prefix:** `172.16.10.0/24`
- **Enable DHCP:** `No`

<img width="713.25" height="352.5" alt="image" src="https://github.com/user-attachments/assets/f0c3de26-9c5c-40db-a218-ec346edd8a32" />

<br>
<br>

-  Enable & attach network adapter 1 to `NAT Network` and select `External_WAN`. Same process for pfSense WAN & Kali.

<br>

<img width="696" height="411" alt="image" src="https://github.com/user-attachments/assets/c05bad67-489e-42d1-bba1-9afa67f8abe0" />

<br>
<br>

Create Secure_LAN:

Typed into each VMs network adapter settings rather than created separately.
-  Enable & attach network adapter 2 to `Internal Network`, type in `Secure_LAN`. Repeat for other LAN devices.
<br>



<img width="696.75" height="414" alt="image" src="https://github.com/user-attachments/assets/97882d27-1f08-4a75-9874-9f7fd1d0dcf1" />

<br><br>



## pfSense Setup

### Download & Extract the Installer ISO
1. Download the pfSense DVD Image (ISO) (`.iso.gz`) from the official [pfSense Download Page](https://pfsense.org/download/).
2. On Linux, extract the compressed `.gz` archive via the terminal to get the raw `.iso` file: <br>

   ```bash
   gzip -d netgate-installer-v1.2-RELEASE-amd64.iso.gz
   ```

### Create the Virtual Machine in VirtualBox
Open Oracle VirtualBox Manager, click New, and configure the VM settings:

* **VM Name:** `pfSense`
* **ISO Image:** Select your extracted `.iso` file
* **OS:** `BSD`
* **OS Distribution:** `FreeBSD`
* **OS Version:** `FreeBSD (64-bit)`

<img width="789.75" height="430.50" alt="image" src="https://github.com/user-attachments/assets/abcf7267-40bc-44ee-994f-4704d9d63756" />

<br>

### Specify Virtual Hardware
* **Base Memory:** `2048 MB` (2 GB)
* **Number of CPUs:** `2`

<img width="792.75" height="430.50" alt="image" src="https://github.com/user-attachments/assets/7df29710-d9b8-4ee5-b648-28de5b996ed1" />

<br>

### Specify Virtual Hard Disk
* **Disk Size:** `20 GB` (Dynamically Allocated Storage)

<br>

<img width="791.25" height="430.5" alt="image" src="https://github.com/user-attachments/assets/8b6cd848-0105-48a2-bf4d-87446a4f976d" />

<br>
<br>


### pfSense Installation & Initial Configuration

Start the VM and wait for it to boot. 

* **Copyright and Distribution Notice:** Select Accept.
* **Welcome Screen:** Highlight Install pfSense and press OK.

<br>

### Interface Assignment & WAN Configuration
> **Note: 1** Open VM Settings > Network in VirtualBox to verify the MAC addresses of your adapters before assigning them.

* **Select WAN Interface:** Choose the interface (`em0`) matching the MAC address of your NAT Network (`External_WAN`) adapter. 
* **Configure WAN (`em0`) Network Mode:**
  * **Interface Mode:** `Static`
  * **IP Address:** `172.16.10.2/24`
  * **Default Gateway:** `172.16.10.1`
  * **DNS Server:** `8.8.8.8`
  * **Use Local Resolver:** `false` (Use remote DNS servers, ignore local DNS) 
* **Select LAN Interface:** Choose the interface (`em1`) matching the MAC address of your Internal Network (`Secure_LAN`) adapter.
* **Confirm Interface Assignment:** Verify both interfaces are mapped correctly and select Continue.
 
<br>


### Edition, Storage & Version Selection
* **Active Subscription Validation:** Select Install CE (Community Edition) (free, open-source version).
* **File System & Partition Scheme:** Select ZFS (GPT).
* **ZFS Virtual Device Configuration:** Select Stripe (no redundancy).
* **Target Disk Selection:** Ensure your VirtualBox virtual hard disk (`da0`) is checked (`[X]`) and press OK to confirm formatting.
* **Software Version:** Select Current Stable Version (2.9.0) and press OK to install the latest security patches, updated FreeBSD base drivers, and bug fixes.

<br>

### Post-Installation Cleanup
1. Once the installation finishes, power off the VM.
2. In VirtualBox, navigate to Settings > Storage.
3. Select the installer `.iso` under Controller: IDE (Optical Drive) and click Remove Disk from Virtual Drive so the VM boots from the virtual hard disk rather than restarting the installer.
4. Start the VM

Below is what the console looks like. Hit `8` to go into shell and ping `www.google.com` to verify you can access the internet.
<img width="966" height="660.75" alt="image" src="https://github.com/user-attachments/assets/ffcf705a-9e35-4c58-a38d-1a304dcb3688" />


<br>
> **Note: 2** We finish setting up pfSense via web based GUI. You will need .

