
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
* **Disk Size:** `20 GB`

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

Below is what the console looks like. Hit `8` to go into shell and ping `www.google.com` to verify you can access the internet via the VirtualBox NAT Network Gateway(`172.16.10.1`).
<br>
<img width="724.5" height="495.5625" alt="image" src="https://github.com/user-attachments/assets/ffcf705a-9e35-4c58-a38d-1a304dcb3688" />


<br>
> **Note: 2** We will finish setting up pfSense via web based GUI. You will need a device with a browser that's inside the same network as the LAN interface. Windows 11 endpoint is the next natural step.

## Windows 11 Workstation Setup

### Download the Installer ISO

Download Windows 11 (multi-edition ISO for x64 devices) from the official [Microsoft Download Page](https://www.microsoft.com/en-us/software-download/windows11).

### Create the Virtual Machine in VirtualBox
Open Oracle VirtualBox Manager, click New, and configure the VM settings:

* **VM Name:** `Windows-11`
* **ISO Image:** Select your `.iso` file
* **OS Edition:** `Windows 11 Home (10.0.26300.9457 / x64 / en-GB)`
* **OS:** `Microsoft Windows`
* **OS Version:** `Windows 11 (64-bit)`

<img width="1404" height="619" alt="image" src="https://github.com/user-attachments/assets/a974ecef-1f18-4e27-9835-840bfea77b9d" />

### Set up unattended guest OS installaion
This is to automate the setup process so you don't have to manually click through each setup screen manually. You will need to enter `username` and `password`. I picked administrator. 

<img width="1406" height="620" alt="image" src="https://github.com/user-attachments/assets/4d690cc5-f4db-4938-bb21-485dd1447c3c" />

### Specify virtual hardware
* **Base Memory:** `4096 MB` (4 GB)
* **Number of CPUs:** `2`

<img width="1403" height="617" alt="image" src="https://github.com/user-attachments/assets/523c33fd-3da4-4466-a374-9cf757a2df3e" />

### Specify Virtual Hard Disk
* **Disk Size:** `50 GB`

<img width="1404" height="617" alt="image" src="https://github.com/user-attachments/assets/177d0f6f-bee0-4f98-873c-56e7bd0d4bd6" />

### Post-Installation Tasks

* When the installation completes, shut down the VM and attach adapter 1 to `Internal Network` `Secure_LAN` in VirtualBox Manager.
* Remove the installer `.iso` from the under Controller: SATA (Optical Drive) and click remove attachment so the VM boots from the virtual hard disk rather than restarting the installer.

### Configure Network Settings

`Start > Settings > Network & Internet > Ethernet > IP assignment`

* **IP address:** 192.168.10.20
* **Subnet Mask:** 255.255.255.0
* **Gateway:** 192.168.10.1
* **Preferred DNS:** 192.168.10.1 (No DC yet for DNS so select pfSense LAN interface)



<br>

## Finish Configuring pfSense via the Web GUI and Set Up Suricata



### 1. Access the pfSense Web GUI

1. Open a web browser and navigate to the pfSense LAN interface:

   `http://192.168.10.1`

2. Log in using the default credentials.
3. Immediately change the default password to something more secure.

### 2. Configure General Settings

**Set the correct timezone**

Navigate to:

`System > General Setup > Localization > Timezone`

Select the appropriate timezone to ensure accurate timestamps in system and security logs.

**Configure WAN reserved networks**

Navigate to:

`Interfaces > WAN > Reserved Networks`

Uncheck:

- `Block private networks and loopback addresses`

This setting blocks traffic originating from private IP address ranges, including:

- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

Our Kali Linux machine uses a private IP address, so we need to ensure this traffic isn't inadvertently blocked by the WAN interface.

### 3. Configure DNS Resolver

Navigate to:

`Services > DNS Resolver > General DNS Resolver Options`

Make sure the DNS Resolver is enabled.

This allows pfSense to act as the upstream DNS server for the Domain Controller.

The intended DNS configuration is:

- The Domain Controller handles internal DNS queries for the Active Directory domain.
- External DNS queries, such as requests for `www.google.com`, are forwarded to the pfSense firewall.
- pfSense resolves these external queries using its upstream DNS configuration.

### 4. Install and Configure Suricata

**Install Suricata**

1. Navigate to `System > Package Manager > Available Packages`.
2. Search for `suricata`.
3. Click **Install**.

Once installation is complete, navigate to:

`Services > Suricata`

**Enable the Emerging Threats Open (ET Open) ruleset**

Suricata relies on rules (also known as signatures) to identify potentially malicious network activity.

1. Enable the **Emerging Threats Open (ET Open)** ruleset.
2. Navigate to the **Updates** section.
3. Update the rules to download the latest available signatures.

**Enable Suricata rules**

Navigate to:

`Services > Suricata > Interface Settings > LAN Categories`

Enable the required rule categories and restart the Suricata service on the LAN interface so the updated rules take effect.

### 5. Configure the Suricata Monitoring Interface

We need to decide which network interface Suricata should monitor.

For this lab, we will use the **LAN interface**.

There are a few reasons for this:

- Suricata will inspect traffic that has already passed through the firewall.
- We will be able to see the actual internal IP address of the affected machine in the alerts.
- This makes it easier to investigate suspicious activity and correlate network alerts with endpoint logs.

For example, if a Windows machine generates suspicious traffic, we want Suricata to identify its actual LAN IP address rather than only seeing the traffic from an upstream network perspective.

### 6. Disable Hardware Checksum Offloading

Suricata requires hardware checksum offloading to be disabled in this pfSense configuration to ensure packet inspection works correctly.

Navigate to:

`System > Advanced > Networking`

Under **Network Interfaces**, disable:

- Hardware Checksum Offloading

A reboot is required for this change to take effect.

### 7. Keep Suricata in IDS Mode

By default, Suricata operates in **IDS (Intrusion Detection System)** mode, meaning it detects suspicious activity and generates alerts without actively blocking the traffic.

We will leave Suricata in IDS mode rather than enabling IPS (Intrusion Prevention System) mode.

<img width="1418" height="170" alt="image" src="https://github.com/user-attachments/assets/97332770-da1f-4ded-aedc-8ed69b63d77d" />

**Reason:**

One of the objectives of this lab is to observe and investigate attacks across multiple security layers.

For example, if Suricata blocks Hydra brute-force attempts at the perimeter, those attempts may never reach the Windows machine.

As a result, we would lose the opportunity to observe failed Windows logon events (Event ID `4625`) and correlate them with the corresponding network activity.

Keeping Suricata in IDS mode allows the traffic to reach the target machine while still generating network-level alerts.



<br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br>




## Suricata IDS Integration (pfSense to Wazuh)

By default, Wazuh expects Suricata logs in JSON (`eve.json`) format, while pfSense forwards Suricata alerts via syslog as plain text. To ingest and alert on pfSense Suricata events in Wazuh:

### 1. Configure pfSense
* Navigated to **Services > Suricata > Interfaces > Edit** and enabled **Send Alerts to System Log**.
* Verified under **Status > System Logs > Settings** that remote logging is enabled (UDP port `514`, BSD RFC 3164 format) and includes **System Events**.

### 2. Configure Wazuh Custom Decoder & Rule
Because pfSense omits the hostname in Suricata syslog headers (causing Wazuh's pre-decoder to misread `suricata` as the hostname), I created a PCRE2 decoder matching `suricata[PID]:` directly in `/var/ossec/etc/decoders/local_decoder.xml`:

```xml
<decoder name="pfsense-suricata">
  <prematch type="pcre2">suricata\[\d+\]:</prematch>
</decoder>

<decoder name="pfsense-suricata-fields">
  <parent>pfsense-suricata</parent>
  <regex type="pcre2">suricata\[\d+\]: \[(\d+:\d+:\d+)\] (.+) \[Classification: ([^\]]*)\] \[Priority: (\d+)\] \{(\w+)\} (\d+\.\d+\.\d+\.\d+):(\d+) -> (\d+\.\d+\.\d+\.\d+):(\d+)</regex>
  <order>id, description, classification, priority, protocol, srcip, srcport, dstip, dstport</order>
</decoder>
```

Added a matching alert rule in `/var/ossec/etc/rules/local_rules.xml`:

```xml
<group name="suricata,pfsense,">
  <rule id="100500" level="7">
    <decoded_as>pfsense-suricata</decoded_as>
    <description>pfSense Suricata Alert: \$(description)</description>
  </rule>
</group>
```

### 3. Verification
* Restarted `wazuh-manager` to apply changes.
* Triggered a test alert from a Windows 11 client using `curl -A "BlackSun" http://example.com`.
* Confirmed alert **Rule ID `100500`** appeared in the Wazuh **Threat Hunting** dashboard with extracted source/destination IPs, ports, and protocol fields.

### HOW ITS DONE

## How the Wazuh Decoder and Rule Work

Wazuh processes incoming logs in a two-step pipeline:
1. **The Decoder (`local_decoder.xml`)** acts like a highlighter. It reads a raw text line and extracts key details (such as source IP, destination IP, and the alert message).
2. **The Rule (`local_rules.xml`)** acts like a security guard. It checks those extracted details, assigns a severity score, and creates the alert visible on the dashboard.

---

### 1. The Decoder (`local_decoder.xml`)

```xml
<decoder name="pfsense-suricata">
  <prematch type="pcre2">suricata\[\d+\]:</prematch>
</decoder>
```

* **`<decoder name="pfsense-suricata">`**: Assigns a unique name to the parent decoder so child decoders and rules can reference it.
* **`<prematch type="pcre2">`**: Instructs Wazuh to scan incoming log lines using standard Perl Compatible Regular Expressions (`pcre2`).
* **`suricata\[\d+\]:`**: Looks for the word `suricata` followed by one or more digits (`\d+`) inside square brackets and a colon (for example, `suricata[46157]:`). If matched, Wazuh identifies the log as a pfSense Suricata event.

```xml
<decoder name="pfsense-suricata-fields">
  <parent>pfsense-suricata</parent>
  <regex type="pcre2">suricata\[\d+\]: \[(\d+:\d+:\d+)\] (.+) \[Classification: ([^\]]*)\] \[Priority: (\d+)\] \{(\w+)\} (\d+\.\d+\.\d+\.\d+):(\d+) -> (\d+\.\d+\.\d+\.\d+):(\d+)</regex>
  <order>id, description, classification, priority, protocol, srcip, srcport, dstip, dstport</order>
</decoder>
```

* **`<parent>pfsense-suricata</parent>`**: Links this field-extraction step to the parent decoder so it only runs on confirmed Suricata logs.
* **`<regex type="pcre2">`**: Matches the layout of a Suricata alert line. Each set of parentheses `( ... )` acts as a capture group that extracts specific text:
  * `\[(\d+:\d+:\d+)\]` captures the rule signature ID (e.g., `1:2008983:10`).
  * `(.+)` captures the alert description (e.g., `ET USER_AGENTS Suspicious User Agent (BlackSun)`).
  * `\[Classification: ([^\]]*)\]` captures the threat classification.
  * `\[Priority: (\d+)\]` captures the priority level (e.g., `1`).
  * `\{(\w+)\}` captures the network protocol inside curly braces (e.g., `TCP`).
  * `(\d+\.\d+\.\d+\.\d+):(\d+)` captures the IP address and port number (first before `->` for the source, and again after `->` for the destination).
* **`<order>`**: Maps each of the 9 captured values in left-to-right order to searchable Wazuh field names (`id`, `description`, `srcip`, etc.).

---

### 2. The Rule (`local_rules.xml`)

```xml
<group name="suricata,pfsense,">
  <rule id="100500" level="7">
    <decoded_as>pfsense-suricata</decoded_as>
    <description>pfSense Suricata Alert: \$(description)</description>
  </rule>
</group>
```

* **`<group name="suricata,pfsense,">`**: Attaches searchable tags (`rule.groups`) so all pfSense or Suricata alerts can be filtered together on the dashboard.
* **`<rule id="100500" level="7">`**:
  * **`id="100500"`**: Sets a unique rule ID. Wazuh reserves IDs `100000`–`120000` for custom user rules so they never conflict with built-in rules.
  * **`level="7"`**: Sets the alert severity on a scale from `0` (ignored) to `15` (critical). Any event at level `3` or higher generates a visible dashboard alert.
* **`<decoded_as>pfsense-suricata</decoded_as>`**: Triggers this rule whenever a log matches the `pfsense-suricata` decoder.
* **`<description>`**: Defines the alert title shown in the dashboard. The `$(description)` variable dynamically inserts the exact alert message captured by the decoder.


<br><br><br><br><br><br><br><br><br><br><br><br><br><br>
<br><br><br><br><br><br><br><br><br><br><br><br><br><br>






















---

## Future Tasks

### 8. Generate a Test Alert

Generate a safe test alert from a host behind pfSense using the [testmynids.org](https://testmynids.org/) project.

The objective is to:

1. Generate test traffic from an internal host.
2. Confirm that Suricata detects the traffic.
3. Verify that the corresponding alert appears in the Suricata logs.

This will confirm that Suricata is running correctly and inspecting LAN traffic.

### 9. Forward pfSense / Suricata Logs to Wazuh

The next step is to configure pfSense to forward its logs to the Wazuh server.

Our Wazuh server has the following IP address:

`192.168.10.30`

The intended syslog destination is:

`192.168.10.30:514`

Navigate to:

`Status > System Logs > Settings`

Configure remote logging as follows:

1. Enable remote logging.
2. Set the remote syslog server to `192.168.10.30:514`.
3. Under **Remote Syslog Contents**, select `Everything`.

This should forward the selected pfSense log categories to Wazuh for centralised monitoring and analysis.

**Note:** Confirm that the Wazuh server is configured to receive remote syslog traffic on UDP/TCP port 514, as appropriate for the transport selected in pfSense. Also verify that the firewall allows the traffic and that the Wazuh syslog integration is configured to process the incoming messages.

### Configure Wazuh server to listen on port 514 for syslog traffic


Wazuh does not listen for external syslog messages by default, we need to manually enable this listener on Wazuh manager before pfSense can send logs to it. 

Add below to `/var/ossec/etc/ossec.conf` on Wazuh manager.

```
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>192.168.10.1</allowed-ips>
</remote>

```
restart wazuh manager service

Confirm server is listening on port 514 with `sudo ss -lunp | grep ':514'`

<img width="1214" height="49" alt="image" src="https://github.com/user-attachments/assets/e18cb9ea-64d1-4211-a56b-c82f9ca42863" />


## Wazuh Setup and Configuration

What is SIEM (optional)
Overview of Wazuh SIEM
Wazuh Architecture
Wazuh installation 
Configuration
Wazuh Agent installation and configuration
Conclusion

We will use Wazuh's [Virtual Machine (VM)](https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html) installation guide to install the pre-built VM image in Open Virtual Appliance (OVA) format. It includes Amazon Linux 2023 operatining system and the Wazuh central components:
* Wazuh manager 4.14.10.
* Filebeat-OSS 7.10.2
* Wazuh indexer 4.14.10
* Wazuh dashboard 4.14.10

### Download & Import the OVA file

Download the file from the official page.

In VirtualBox Manager: `File > Import Appliance`.
Select `.ova` file.
It's pre-configured so no need to select any other settings. Click finish. 

### Set the Graphics Controller
Wazuh recommnds setting the Graphics Controller to `VMSVGA` in VirtualBox to avoid freezes.
* `Settings > Display` - Switch from Basic to Expert mode.
* From the Graphics controller dropdown, select the `VMSVGA` option.

### Change the network adapter

* `Settings > Network`
* Under adapter 1, select Enable Network Adapter
* Attach it to `Internal Network`
* Set name: `Secure_LAN`


### Start Wazuh and configure its IP address

Log in using the default credentials: `wazuh-user/wazuh`.

### Configure network files in Amazon Linux

open network configuration file eth0:
* `sudo nano /etc/systemd/network/20-eth0.network`
[Match]
Name=eth0

[Network]
Address=192.168.10.30/24
Gateway=192.168.10.1
DNS=192.168.10.1

restart network service
* `sudo systemctl restart systemd-networkd`

* browse to http://192.168.10.30 on Windows 11 to Access Wazuh dashboard. `admin/admin`.


## Wazuh agent installation and configuration

What is Wazuh Agent?
* Wazuh agent is a lightweight service that runs on endpoints, collects security data and sends them to Wazuh server for processing and analysis.
Key features include:
* Log collector
* File Integrity Monitoring (FIM)
* Malware detection
* Security Configuration Asseessment (SCA)

### Deploying the agent via Wazuh dashboard

* Navigate to `Agents management > Summary` and click `Deploy new agent`
* Select Windows, enter IP address of the wazuh server, agent name will be PC1
* Go back to powershell on Windows 11 and execute the following command:

```

Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.10-1.msi -OutFile $env:tmp\wazuh-agent; msiexec.exe /i $env:tmp\wazuh-agent /q WAZUH_MANAGER='192.168.10.30' WAZUH_AGENT_NAME='PC1'
```

* Start the agent with `NET START Wazuh`
Verify agent connection in wazuh dashboard:
<img width="1778" height="484" alt="image" src="https://github.com/user-attachments/assets/b7789023-af44-4593-b10e-6619f698338b" />

* All agent files are stored in `C:\Program Files (x86)\ossec-agent`



## File Integrity Monitoring (FIM) integration

FIM tracks file and directory modifications in real time and generates alerts when suspicious activity is detected. 

* Locate and edit `ossec.conf` file on windows 11 endpoint.
* find <syscheck> block and insert:

```

<directories check_all="yes" realtime="yes" report_changes="yes">C:\Users\Public\Downloads</directories>
```

* `realtime` watches continuously rather than on a scan interval
* `report_changes` logs what changed inside a file, not just that it changed.

* Restart wazuh agent to apply changes.
* Create, edit and delete a text file inside the downloads folder.
* Navigate to `File Integrity Monitoring > Events` to Validate wazuh alerts show file creation, modification and deletetion:
<img width="1772" height="382" alt="image" src="https://github.com/user-attachments/assets/e5e3addf-9040-4754-a80e-bfa58504a1f4" />



## VirusTotal Integration 

VirusTotal is a threat intelligence platform owned by google that a free online security service owned by Google that analyzes suspicious files, URLs, domains, and IP addresses for malware and phishing threats. VirusTotal Checks submissions against more than 70 different antivirus engines and online scanners simultaneously. <br> 
By integrating VirusTotal into Wazuh, we can automatically enrich raw alerts with reputation data, making investigations faster and more accurate. 


* VirusTotal checks every directory monitored by Wazuh File Integrity Monitoring (FIM), though only `C:\Users\Public\Downloads` is currently monitored in real time.
* Note: Files/URLs uploaded to VirusTotal are public so don't upload sensitive data.
* We will use VirusTotal API to automate lookups and integrate it with Wazuh. This integration enables Wazuh to automatically query VirusTotal when new files are detected on endpoints. 

### Obtain a VirusTotal API key and configure in Wazuh

* sign up at virustotal.com and generate a free API key. free-tier limit allows roughly 4 requests per minute and 500 a day. dropping multiple files at once will hit that ceiling so be aware.
* Navigate to Wazuh server `ossec.conf` file (not the agent) and add:

```
<integration>
  <name>virustotal</name>
  <api_key>YOUR_KEY_HERE</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```
* restart wazuh service

### Validate with EICAR test file

EICAR is a harmless string the antivirus industry universally treats as a test “virus.” It is Designed to trigger alerts: All compliant antivirus and anti-malware programs are programmed to recognize its specific signature and treat it exactly as if it were a dangerous threat (blocking, quarantining, or alerting).

* Download the file from the [official page](https://www.eicar.org/download-anti-malware-testfile/) and save it inside the monitored folder `C:\Users\Public\Downloads`.
* Navigate to `Threat intelligence > Threat Hunting > Events` to validate File Integrity Monitoring and VirusTotal integration are both working.
* The below image shows that EICAR malware test succeeded. Wazuh detected the file download in real time and flagged it via VirusTotal with 59 out of 61 security engines identifying it as malicious.

<img width="1700" height="252" alt="image" src="https://github.com/user-attachments/assets/0f2f2a9d-aa3e-4266-bc1b-9713991d3f03" />
<img width="926" height="951" alt="image" src="https://github.com/user-attachments/assets/b0ac8bc5-4386-417c-bf01-5e8bdd7d0768" />
<img width="1803" height="514" alt="image" src="https://github.com/user-attachments/assets/7bc05b00-9ff4-4d0b-ac2e-aff072598261" />



Logic behind the hash lookup: When a new file is saved in the monitored folder, the Wazuh agent calculates its cryptographic hashes (MD5, SHA1, and SHA256) locally on the endpoint and sends those fingerprints to the Wazuh Manager without ever uploading the actual file. This triggers a File Integrity Monitoring alert, prompting the manager's wazuh-integratord daemon to pass the file hash to the VirusTotal API via virustotal.py. VirusTotal checks if that exact hash matches known malware in its database and returns a JSON report, which Wazuh then parses to generate a high-severity detection alert in the dashboard.



<br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br>

---

## Domain controller setup and configuration
Confirm FIM and VirusTotal are working cleanly before joining the domain. 

We will join our windows 11 workstation to domain controller for DNS queries, to mimic real corporate network (so the brute force attempts generate domain authentication events. this is what a SOC analyst would normally triage day to day, investigating local account brute force is rare in an enterprise environment) and deploy lockout policies, sysmon & XML configuration file via group policy (GPO). XML configurtion file is a rulebook that tells sysmon what to monitor and what to ignore. 

Download server iso from [official page](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022).

### VirtualBox
* 4gb (4096 mb) RAM
* 2vCpus
* 50gb storage


 ### Domain Controller setup and configure 
* configure IP settings.
* promote to DC: Server manager > add roles > active directory domain services. run promotion wizard.
* In DNS manager, Configure its forwarder to point at pfSense’s LAN interface (192.168.10.1)
* Domain-join Windows 11: change its primary DNS to the DC’s IP first (it has to be able to find the domain controller before it can join it), then System > Domain, enter the domain name, reboot.
* Create a new OU and Create a user account in AD. i created Bob.
* account lockout policy via group policy: 
* Group Policy Management (gpmc.msc) > Right-click Default Domain Policy and select edit. navigate to Computer Configuration > Policies > Windows Settings > Security Settings > Account Policies > Account Lockout Policy. set account lockout threshold to 5. apply and verify: gpupdate /force on the domain controller. type incorrect password 5 times on windows 11 workstation and check security log on domain controller for event id 4740. 

###Deploy Sysmon + XML config to windows 11 workstation

download [sysmon]([https://live.sysinternals.com/](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)) and extract to C:\sysmon
* once installed on a system, remains resident across system reboots to monitor and log system activity to the Windows event log. It provides detailed information about process creations, network connections, and changes to file creation time
* Install: sysmon64 -i [<configfile>] . the XML config file is a rulebook. it defines what to log and what to exclude. we will use SwiftOnSecurity sysmonconfig-export.xml rulebook from SwiftOnSecurity [github repo](https://github.com/SwiftOnSecurity/sysmon-config/blob/master/README.md). extract to same folder. raw sysmon output is mostly noise, so a fine-tuned config like this is what makes it usable. 
* Run .\Sysmon.exe -accepteula -i sysmonconfig-export.xml with admin rights.
* Run .\Sysmon.exe -c sysmonconfig-export.xml to update configuration file.
* events are stored in `Applications and Services Logs/Microsoft/Windows/Sysmon/Operational`
* point wazuh agent at the sysmon log. this steps gets sysmon data into the SIEM rather than just sitting locally on windows 11 agent's ossec.conf file.

```
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

* create a test event: Run a command that Wazuh's Sysmon ruleset flags as suspicious (for example, an encoded PowerShell command or whoami reconnaissance). Open PowerShell on PC1 and run:
* `powershell.exe -EncodedCommand dwBoAG8AYQBtAGkA` (This safely decodes to whoami.)

<img width="1697" height="266" alt="image" src="https://github.com/user-attachments/assets/69769bfa-2802-432c-bad7-ad79fec9b620" />
• Sysmon Event ID 1: Powershell.exe spawned a powershell process which executed a base64ecoded command (the -EncodedCommand test).
• Sysmon Event ID 11: Executable file dropped in folder commonly used by malware (PowerShell creating a temporary test policy/script file in AppData\Local\Temp when executing).



<br><br><br><br><br><br><br><br><br><br><br><br><br><br><br>
<br><br><br><br><br><br><br><br><br><br><br><br><br><br><br>
<br><br><br><br><br><br><br><br><br><br><br><br><br><br><br>

# SIEM Detection Lab: Build Log

This log covers how I built the lab, the decisions behind each setting, and the problems I hit along the way. Problems and workarounds are marked **Gotcha**.

**Stack:** VirtualBox, pfSense (firewall, router, Suricata), Wazuh SIEM, Windows 11 endpoint, Windows Server 2022 Domain Controller, Kali Linux (attacker, added later).

> **Legend for notes to self (delete before publishing):**
> 📸 = screenshot worth adding
> ✏️ = detail I should fill in
> ⚠️ = check this is accurate

---

## 1. Network Design

Two zones, with pfSense sitting between them:

- **External_WAN:** the untrusted internet / attacker network.
- **Secure_LAN:** the trusted internal corporate network.

pfSense has three roles:
- **Firewall:** filters traffic between WAN and LAN.
- **Router:** routes traffic between the two networks.
- **Suricata IDS:** monitors traffic for suspicious activity.

<img width="752" height="677" alt="Untitled Diagram (3)" src="https://github.com/user-attachments/assets/cde3df03-30c5-4c84-9c8e-28532210bf6a" />

### VirtualBox networking modes

Based on Oracle's [networking modes documentation](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html#networkingmodes):

- **NAT Network (`External_WAN`):** a shared virtual network where VMs can talk to each other and reach the internet through VirtualBox's NAT engine. Hosts Kali and the pfSense WAN interface.
- **Internal Network (`Secure_LAN`):** a private, isolated virtual switch. VMs can talk to each other but not to the host or external networks, so they depend on pfSense for everything else. Hosts Windows Server, Windows 11, Wazuh, and the pfSense LAN interface.

| Feature | External_WAN | Secure_LAN |
| :--- | :--- | :--- |
| **VirtualBox mode** | NAT Network | Internal Network |
| **Purpose** | Simulated external network | Simulated corporate network |
| **Internet access** | Yes (via VirtualBox NAT engine) | No direct access; routed through pfSense |
| **Host-to-VM access** | Blocked by default (needs port forwarding) | None |
| **Contains** | Kali Linux, pfSense WAN | Windows Server, Windows 11, Wazuh, pfSense LAN |
| **Security zone** | Untrusted | Trusted |

### VM network adapters

| Virtual Machine | Adapter 1 | Adapter 2 |
| :--- | :--- | :--- |
| **pfSense** | NAT Network (`External_WAN`) | Internal Network (`Secure_LAN`) |
| **Kali Linux** | NAT Network (`External_WAN`) | Disabled |
| **Windows Server** | Internal Network (`Secure_LAN`) | Disabled |
| **Windows 11** | Internal Network (`Secure_LAN`) | Disabled |
| **Wazuh SIEM** | Internal Network (`Secure_LAN`) | Disabled |

### IP addressing plan

Two private RFC 1918 subnets:

- **External_WAN:** `172.16.10.0/24`
- **Secure_LAN:** `192.168.10.0/24`

**External_WAN: 172.16.10.0/24**

| Device / Interface | IP Address | Gateway | DNS |
| :--- | :--- | :--- | :--- |
| VirtualBox NAT Network gateway | `172.16.10.1` | n/a | n/a |
| pfSense WAN | `172.16.10.2` | `172.16.10.1` | `8.8.8.8` |
| Kali Linux | `172.16.10.10` | `172.16.10.1` | `8.8.8.8` |

**Secure_LAN: 192.168.10.0/24**

| Device / Interface | IP Address | Gateway | DNS |
| :--- | :--- | :--- | :--- |
| pfSense LAN | `192.168.10.1` | n/a (it is the gateway) | pfSense DNS Resolver (own lookups only) |
| Windows Server (Domain Controller) | `192.168.10.10` | `192.168.10.1` | `192.168.10.10` |
| Windows 11 | `192.168.10.20` | `192.168.10.1` | `192.168.10.10` |
| Wazuh SIEM | `192.168.10.30` | `192.168.10.1` | `192.168.10.10` ⚠️ |

> ⚠️ Later in this log the Wazuh VM's network file sets `DNS=192.168.10.1` (pfSense), not the DC. Either update the table to match, or change the Wazuh config to point at the DC. Same question for Windows 11: the table shows its final DNS (the DC), but it starts on pfSense until the domain join.

**Design decisions**
- **Static IPs, no DHCP:** firewalls and servers should never change address, so the DC doesn't hand out leases.
- **DNS split:** Secure_LAN devices use the DC for internal DNS and domain joins. The DC forwards external queries to pfSense, because Secure_LAN has no other route out.
- **NAT Network gateway:** VirtualBox assigns the first address (`.1`) to the NAT Network gateway. Confirm with `VBoxManage natnetwork list`.

### Creating the networks in VirtualBox

**External_WAN:** `Network > NAT Networks > Create`
- **Name:** `External_WAN`
- **IPv4 prefix:** `172.16.10.0/24`
- **Enable DHCP:** No (static addressing only)

<img width="713.25" height="352.5" alt="image" src="https://github.com/user-attachments/assets/f0c3de26-9c5c-40db-a218-ec346edd8a32" />

Attach adapter 1 of pfSense and Kali to `NAT Network` > `External_WAN`.

**Secure_LAN:** Internal Networks aren't created separately in VirtualBox. Type `Secure_LAN` into the adapter settings of each LAN VM (pfSense adapter 2, Windows Server, Windows 11, Wazuh) and they join the same virtual switch.

<!-- Removed as noise (adapter settings screenshots, restore if wanted):
<img width="696" height="411" alt="image" src="https://github.com/user-attachments/assets/c05bad67-489e-42d1-bba1-9afa67f8abe0" />
<img width="696.75" height="414" alt="image" src="https://github.com/user-attachments/assets/97882d27-1f08-4a75-9874-9f7fd1d0dcf1" />
-->

---

## 2. pfSense Installation

1. Download the pfSense installer (`.iso.gz`) from the [official download page](https://pfsense.org/download/).
2. Extract it on Linux:
   ```bash
   gzip -d netgate-installer-v1.2-RELEASE-amd64.iso.gz
   ```

**VM settings**

| Setting | Value |
| :--- | :--- |
| Name | `pfSense` |
| OS / Distribution / Version | BSD / FreeBSD / FreeBSD (64-bit) |
| RAM | 2048 MB |
| CPUs | 2 |
| Disk | 20 GB |

<!-- Removed as noise (VM hardware wizard screenshots, restore if wanted):
<img width="789.75" height="430.50" alt="image" src="https://github.com/user-attachments/assets/abcf7267-40bc-44ee-994f-4704d9d63756" />
<img width="792.75" height="430.50" alt="image" src="https://github.com/user-attachments/assets/7df29710-d9b8-4ee5-b648-28de5b996ed1" />
<img width="791.25" height="430.5" alt="image" src="https://github.com/user-attachments/assets/8b6cd848-0105-48a2-bf4d-87446a4f976d" />
-->

**Installer choices** (accept the licence, then Install pfSense)

- **Interface assignment:** check the MAC addresses under VM Settings > Network first, so you assign the right adapter to the right role.
  - WAN = `em0` (the `External_WAN` adapter)
  - LAN = `em1` (the `Secure_LAN` adapter)
- **WAN configuration:** static, `172.16.10.2/24`, gateway `172.16.10.1`, DNS `8.8.8.8`, local resolver off (use the remote DNS server, not pfSense's own).
- **Edition:** Community Edition (free).
- **Filesystem:** ZFS (GPT), Stripe (no redundancy needed in a lab), target disk `da0`.
- **Version:** Current Stable (2.9.0).

**After install**
1. Power off, then remove the installer ISO under `Settings > Storage > Controller: IDE` so the VM boots from disk instead of restarting the installer.
2. Start the VM. From the console menu press `8` for a shell and run `ping www.google.com` to confirm internet access through the NAT gateway (`172.16.10.1`).


<img width="724.5" height="495.5625" alt="image" src="https://github.com/user-attachments/assets/ffcf705a-9e35-4c58-a38d-1a304dcb3688" />

The rest of pfSense is configured through the web GUI, which needs a browser *inside* Secure_LAN. The Windows 11 endpoint is the natural next step.

---

## 3. Windows 11 Workstation

1. Download the Windows 11 multi-edition ISO (x64) from [Microsoft](https://www.microsoft.com/en-us/software-download/windows11).
2. Create the VM:

| Setting | Value |
| :--- | :--- |
| Name | `Windows-11` |
| OS / Version | Microsoft Windows / Windows 11 (64-bit) |
| Edition | Windows 11 Home (Upgrade to Pro later for domain join) |
| RAM | 4096 MB |
| CPUs | 2 |
| Disk | 50 GB |


I used VirtualBox's unattended install to skip the setup screens (username `administrator`).

<!-- Removed as noise (VM wizard screenshots, restore if wanted):
<img width="1404" height="619" alt="image" src="https://github.com/user-attachments/assets/a974ecef-1f18-4e27-9835-840bfea77b9d" />
<img width="1406" height="620" alt="image" src="https://github.com/user-attachments/assets/4d690cc5-f4db-4938-bb21-485dd1447c3c" />
<img width="1403" height="617" alt="image" src="https://github.com/user-attachments/assets/523c33fd-3da4-4466-a374-9cf757a2df3e" />
<img width="1404" height="617" alt="image" src="https://github.com/user-attachments/assets/177d0f6f-bee0-4f98-873c-56e7bd0d4bd6" />
-->

**After install**
- Shut down, attach adapter 1 to Internal Network `Secure_LAN`, and remove the installer ISO from `Controller: SATA`.
- Set a static IP under `Settings > Network & Internet > Ethernet > IP assignment`:
  - IP `192.168.10.20`, mask `255.255.255.0`, gateway `192.168.10.1`
  - DNS `192.168.10.1` (pfSense, because there is no DC yet. This changes to the DC's IP before the domain join.)

---

## 4. pfSense Web GUI and Suricata

Browse to `http://192.168.10.1` from Windows 11, log in with the default credentials, and **change the default password immediately**.

### General settings
- **Timezone:** `System > General Setup > Localization`. Set it correctly so log timestamps line up across pfSense, Suricata and Wazuh.
- **DNS Resolver:** `Services > DNS Resolver`. Make sure it is enabled so pfSense can act as the upstream resolver for the Domain Controller (the DC handles AD DNS; external lookups like `www.google.com` get forwarded to pfSense).

> 📸 **SCREENSHOT:** DNS Resolver settings page. This is a key config and ties into the DNS gotcha below.

> **Gotcha: WAN drops traffic from private IPs**
> pfSense blocks private (RFC 1918) source addresses on the WAN interface by default. External_WAN uses `172.16.10.0/24`, which falls inside the blocked `172.16.0.0/12` range, so Kali's traffic was being silently dropped at the firewall.
> **Fix:** `Interfaces > WAN > Reserved Networks`, untick **Block private networks and loopback addresses**. (This is fine here because the WAN is a simulated attacker network. On a real internet-facing firewall you would leave it on.)
> ✏️ ADD: what you saw when it was broken (failed ping/scan from Kali, or a firewall log entry for the blocked packet).

### Installing Suricata
Installed via the Package Manager (`System > Package Manager > Available Packages > suricata`). Then under `Services > Suricata`:
- Enabled the **Emerging Threats Open (ET Open)** ruleset and ran an update to pull the latest signatures.
- Enabled the required rule categories under `Interface Settings > LAN Categories` and restarted Suricata on the LAN interface so they take effect.

<img width="1422" height="1022" alt="image" src="https://github.com/user-attachments/assets/8399ef62-14a5-4af5-80f8-68852fb48be5" />


### Decision: monitor the LAN interface
Suricata listens on **LAN**, not WAN, because:
- It inspects traffic that has already passed the firewall, so alerts reflect what actually reached the internal network.
- Alerts show the real internal IP of the affected machine, which makes it easy to correlate with endpoint logs.

### Decision: stay in IDS mode, not IPS
Suricata runs as an **IDS** (detect and alert, no blocking). One goal of the lab is to observe attacks at several layers. If Suricata blocked a Hydra brute-force at the perimeter, the attempts would never reach the Windows machine and I would lose the failed-logon events (Event ID `4625`) that I want to correlate with the network alerts.

<img width="1418" height="170" alt="image" src="https://github.com/user-attachments/assets/97332770-da1f-4ded-aedc-8ed69b63d77d" />

> **Gotcha: hardware checksum offloading**
> Suricata does not inspect packets correctly while checksum offloading is enabled, because virtual NICs hand it packets with checksums not yet calculated, which leads to missed or bogus detections.
> **Fix:** `System > Advanced > Networking`, tick **Disable hardware checksum offloading**, then reboot.

---

## 5. Wazuh SIEM Installation

I used Wazuh's pre-built [VM image (OVA)](https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html). It runs Amazon Linux 2023 and includes the Wazuh manager 4.14.10, Filebeat-OSS 7.10.2, the Wazuh indexer 4.14.10, and the Wazuh dashboard 4.14.10.

1. Import via `File > Import Appliance` (no settings need changing).
2. **Graphics controller:** set to `VMSVGA` (Settings > Display > Expert mode). Wazuh recommends this to avoid freezes.
3. **Network:** enable adapter 1, attach to Internal Network `Secure_LAN`.
4. Start the VM and log in with the default credentials (`wazuh-user` / `wazuh`).

**Set a static IP** in `/etc/systemd/network/20-eth0.network`:

```ini
[Match]
Name=eth0

[Network]
Address=192.168.10.30/24
Gateway=192.168.10.1
DNS=192.168.10.1
```

Apply with `sudo systemctl restart systemd-networkd`.

Open the dashboard from Windows 11 at `https://192.168.10.30` (default login `admin` / `admin`)

---

## 6. Wazuh Agent on Windows 11

The agent is a lightweight service on the endpoint that collects logs, monitors file integrity (FIM), detects malware, and runs security configuration assessments, then ships everything to the Wazuh manager.

1. In the dashboard: `Agents management > Summary > Deploy new agent`. Choose Windows, enter the manager IP, and name the agent `PC1`.
2. Run the generated command in PowerShell on Windows 11:
   ```powershell
   Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.10-1.msi -OutFile $env:tmp\wazuh-agent; msiexec.exe /i $env:tmp\wazuh-agent /q WAZUH_MANAGER='192.168.10.30' WAZUH_AGENT_NAME='PC1'
   ```
3. Start it with `NET START Wazuh` and confirm the agent shows as active.

Agent files live in `C:\Program Files (x86)\ossec-agent`.

<img width="1778" height="484" alt="image" src="https://github.com/user-attachments/assets/b7789023-af44-4593-b10e-6619f698338b" />

(*Agent active in Wazuh dashboard*)


---


## 7. pfSense Suricata Alerts into Wazuh

**Goal:** get Suricata alerts from pfSense into the Wazuh dashboard.

**Problem:** Wazuh expects Suricata logs as JSON (`eve.json`), but pfSense forwards them over syslog as plain text. That needs a custom decoder and rule.

### Step 1: Make Wazuh listen for syslog

> **Gotcha: Wazuh doesn't accept syslog by default.**
> pfSense's logs went nowhere until I enabled a syslog listener on the Wazuh manager.

Add this to `/var/ossec/etc/ossec.conf` on the manager:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>192.168.10.1</allowed-ips>
</remote>
```

`allowed-ips` is restricted to the pfSense LAN address so no other host can inject logs. Restart `wazuh-manager`, then confirm the listener with `sudo ss -lunp | grep ':514'`.

<img width="1214" height="49" alt="image" src="https://github.com/user-attachments/assets/e18cb9ea-64d1-4211-a56b-c82f9ca42863" />

### Step 2: Forward logs from pfSense

- `Services > Suricata > Interfaces > Edit`: enable **Send Alerts to System Log**.
- `Status > System Logs > Settings`: enable remote logging to `192.168.10.30:514` (UDP, BSD RFC 3164 format) and include **System Events**.

### Step 3: Write the custom decoder and rule

> **Gotcha: pfSense omits the hostname in Suricata syslog headers.**
> Wazuh's pre-decoder then mistakes `suricata` for the hostname, so none of the built-in decoders matched. I wrote a PCRE2 decoder that matches on `suricata[PID]:` directly.

**Before: confirming the problem**

This is what the raw syslog messages look like when they arrive on the Wazuh manager, before any decoder processes them:

<img width="1922" height="80" alt="image" src="https://github.com/user-attachments/assets/d8b2f2a1-f5e5-4440-8e0d-49a5b6f0f141" />

Pass a raw log line to `wazuh-logtest` to see how Wazuh handles it:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

Result: the built-in ruleset missed the log entirely (**No decoder matched**), so Suricata syslog events were being dropped without generating alerts.

<img width="1512" height="282" alt="image" src="https://github.com/user-attachments/assets/0e49b9de-dc9d-4f36-b1c6-80cf260a5287" />

**The fix**

`/var/ossec/etc/decoders/local_decoder.xml`:

```xml
<decoder name="pfsense-suricata">
  <prematch type="pcre2">suricata\[\d+\]:</prematch>
</decoder>

<decoder name="pfsense-suricata-fields">
  <parent>pfsense-suricata</parent>
  <regex type="pcre2">suricata\[\d+\]: \[(\d+:\d+:\d+)\] (.+) \[Classification: ([^\]]*)\] \[Priority: (\d+)\] \{(\w+)\} (\d+\.\d+\.\d+\.\d+):(\d+) -> (\d+\.\d+\.\d+\.\d+):(\d+)</regex>
  <order>id, description, classification, priority, protocol, srcip, srcport, dstip, dstport</order>
</decoder>
```

`/var/ossec/etc/rules/local_rules.xml`:

```xml
<group name="suricata,pfsense,">
  <rule id="100500" level="7">
    <decoded_as>pfsense-suricata</decoded_as>
    <description>pfSense Suricata Alert: $(description)</description>
  </rule>
</group>
```

Restart `wazuh-manager` to load the new decoder and rule.

**After: confirming the fix**

Re-run the same raw log through `wazuh-logtest`. The decoder now maps `srcip`, `srcport`, `dstip`, `dstport`, `protocol`, `id`, `classification`, `priority` and `description`, which makes those fields searchable and usable in dashboards.

<img width="1512" height="725" alt="image" src="https://github.com/user-attachments/assets/51936d10-2db6-4054-aba7-58e7e126f201" />

### How the decoder and rule work
Wazuh processes logs in two steps. The **decoder** reads a raw line and extracts fields (IPs, ports, message). The **rule** evaluates those fields, assigns a severity, and raises the alert.

**Decoder**
- `prematch`: scans each log line with PCRE2 for `suricata[<digits>]:` (e.g. `suricata[46157]:`). A match identifies it as a pfSense Suricata event.
- Child decoder `pfsense-suricata-fields`: only runs on lines the parent matched, and uses capture groups to pull out:
  - the signature ID (e.g. `1:2008983:10`)
  - the alert description (e.g. `ET USER_AGENTS Suspicious User Agent (BlackSun)`)
  - the classification and priority
  - the protocol
  - source and destination IP:port
- `<order>` maps the 9 captured values, left to right, to searchable Wazuh field names.

**Rule**
- `group`: tags alerts (`suricata`, `pfsense`) so they can be filtered together on the dashboard.
- `id="100500"`: Wazuh reserves IDs `100000` to `120000` for custom rules, so there are no clashes with built-in ones.
- `level="7"`: severity on a 0 (ignored) to 15 (critical) scale. Level 3 and above shows on the dashboard.
- `decoded_as`: fires whenever the `pfsense-suricata` decoder matched.
- `$(description)`: inserts the alert message the decoder captured into the alert title.

### Step 4: Verify the network detection pipeline

Trigger a test alert from Windows 11 or the Server:

```bash
curl -A "BlackSun" http://example.com
```

Suricata flags the suspicious `BlackSun` user agent. Then follow the alert through each stage of the pipeline.

**1. Suricata detects it:** `Services > Suricata > Alerts`

<img width="1720" height="177" alt="image" src="https://github.com/user-attachments/assets/f4956885-355c-436a-a560-4dde7205712a" />

**2. pfSense logs it and forwards it:** `Status > System Logs > System > General`

<img width="1718" height="181" alt="image" src="https://github.com/user-attachments/assets/0bb4afef-bb42-4792-a37c-1eaf5fd47208" />

**3. Wazuh raises rule 100500:** Threat Hunting dashboard

<img width="1651" height="228" alt="image" src="https://github.com/user-attachments/assets/060e0578-3010-48aa-afe2-d44fe6001d95" />

**4. The expanded alert shows the extracted fields:**

<img width="960" height="832" alt="image" src="https://github.com/user-attachments/assets/049b2451-19f1-4aa6-aea3-a5e5b75f163d" />
<img width="964" height="790" alt="image" src="https://github.com/user-attachments/assets/6575610b-7cb5-4212-94bf-269d90390fd9" />

The alert is visible at every layer: Suricata, the pfSense system log, and Wazuh.

---

---

## 8. File Integrity Monitoring (FIM)

FIM tracks file and directory changes in real time and alerts on suspicious activity.

On the Windows 11 agent, edit `ossec.conf` and add this inside the `<syscheck>` block:

```xml
<directories check_all="yes" realtime="yes" report_changes="yes">C:\Users\Public\Downloads</directories>
```

- `realtime`: watches continuously instead of waiting for a scheduled scan.
- `report_changes`: records *what* changed inside a file, not just that it changed.

Restart the agent, then create, edit and delete a text file in that folder. Under `File Integrity Monitoring > Events`, confirm the creation, modification and deletion events appear.

<img width="1772" height="382" alt="image" src="https://github.com/user-attachments/assets/e5e3addf-9040-4754-a80e-bfa58504a1f4" />

---

## 9. VirusTotal Integration

[VirusTotal](https://www.virustotal.com) (owned by Google) checks files, URLs, domains and IPs against 70+ antivirus engines and scanners. Integrating it with Wazuh enriches FIM alerts with reputation data automatically, which speeds up triage.

> Anything uploaded to VirusTotal is public. Don't submit sensitive files.

### Setup
1. Create a free VirusTotal account and generate an API key.
2. Add this to `ossec.conf` on the **Wazuh server** (not the agent):
   ```xml
   <integration>
     <name>virustotal</name>
     <api_key>YOUR_KEY_HERE</api_key>
     <group>syscheck</group>
     <alert_format>json</alert_format>
   </integration>
   ```
3. Restart the Wazuh manager.

> **Gotcha: free-tier rate limit**
> The free API allows roughly 4 requests per minute and 500 per day. Dropping several files into the monitored folder at once will hit the ceiling and some lookups will fail.

VirusTotal is triggered by FIM (`syscheck` group), so it covers every FIM-monitored directory. Only `C:\Users\Public\Downloads` is monitored in real time for now.

### How the lookup works
When a file lands in the monitored folder, the agent computes its MD5, SHA1 and SHA256 hashes locally and sends **only the hashes** to the manager (the file itself is never uploaded). The FIM alert prompts the manager's `wazuh-integratord` daemon to pass the hash to the VirusTotal API via `virustotal.py`. VirusTotal checks whether that hash matches known malware and returns a JSON report, which Wazuh parses into a high-severity alert.

### Validation with the EICAR test file
[EICAR](https://www.eicar.org/download-anti-malware-testfile/) is a harmless string that every compliant antivirus engine treats as a test "virus", so it safely triggers detection.

1. Download it into `C:\Users\Public\Downloads`.
2. Check `Threat Intelligence > Threat Hunting > Events`.

**Result:** Wazuh detected the download in real time, and VirusTotal flagged it, with 59 of 61 engines marking it malicious. This confirms FIM and the VirusTotal integration work end to end.

<img width="1700" height="252" alt="image" src="https://github.com/user-attachments/assets/0f2f2a9d-aa3e-4266-bc1b-9713991d3f03" />
<img width="926" height="951" alt="image" src="https://github.com/user-attachments/assets/b0ac8bc5-4386-417c-bf01-5e8bdd7d0768" />
<img width="1803" height="514" alt="image" src="https://github.com/user-attachments/assets/7bc05b00-9ff4-4d0b-ac2e-aff072598261" />

---

## 10. Domain Controller and Active Directory

**Why:** brute-force attempts against a domain account generate domain authentication events, which is what a SOC analyst actually triages day to day. Local-account brute force is rare in an enterprise. I also want to deploy lockout policies from the DC.

> Confirm FIM and VirusTotal work cleanly *before* joining the domain, so any later breakage is easy to isolate.

**VM:** Windows Server 2022 ([evaluation ISO](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022)), 4096 MB RAM, 2 vCPUs, 50 GB disk, static IP `192.168.10.10`.

### Setup
1. **Promote to DC:** Server Manager > Add Roles > Active Directory Domain Services, then run the promotion wizard.
2. **DNS forwarder:** in DNS Manager, point the forwarder at pfSense's LAN interface (`192.168.10.1`). This is the DNS split from section 1: the DC answers for the domain, pfSense answers for everything else.

<img width="603" height="711" alt="image" src="https://github.com/user-attachments/assets/1c9abf76-d0b9-4bd7-a233-6abcff191866" />

4. **Create an OU and a test user** (`Bob`).
5. **Join Windows 11 to the domain:** first set its primary DNS to the DC (`192.168.10.10`), because it has to find the DC before it can join. Then `System > Domain`, enter the domain name and reboot.

> 📸 **SCREENSHOT:** the DC's DNS forwarder config pointing at `192.168.10.1`, and/or `nslookup` from Windows 11 resolving both the domain and an external name. This is the DC/pfSense DNS split that tripped me up.

> ✏️ ADD (gotcha): what failed before the DNS was set correctly (e.g. "domain could not be contacted" error on the join).

### Account lockout policy (GPO)
1. Open `gpmc.msc`, right-click **Default Domain Policy** and choose Edit.
2. Go to `Computer Configuration > Policies > Windows Settings > Security Settings > Account Policies > Account Lockout Policy`.
3. Set **Account lockout threshold** to `5`.
4. Run `gpupdate /force` on the DC.
5. **Test:** enter a wrong password 5 times for Bob on Windows 11, then check the DC's Security log for Event ID `4740` (account lockout).

> 📸 **SCREENSHOT:** the Event ID 4740 lockout event on the DC. Good proof the policy works and it feeds the brute-force detection later.

---

## 11. Sysmon on Windows 11

Sysmon stays resident across reboots and logs detailed system activity to the Windows event log: process creation, network connections, file creation time changes, and more.

**Config choice:** raw Sysmon output is mostly noise, so I use the [SwiftOnSecurity `sysmonconfig-export.xml`](https://github.com/SwiftOnSecurity/sysmon-config) rulebook, which defines what to log and what to exclude.

### Install
1. Download [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) and extract it, plus the config file, to `C:\sysmon`.
2. From an admin prompt:
   ```powershell
   .\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
   ```
3. To update the config later: `.\Sysmon64.exe -c sysmonconfig-export.xml`

> ⚠️ Your intro says Sysmon and the config are deployed via GPO, but the steps above are a manual install on PC1. Either reword the intro or add the GPO steps once you've done them.

Events are written to `Applications and Services Logs/Microsoft/Windows/Sysmon/Operational`.

### Ship Sysmon logs to Wazuh
Without this step the data just sits locally on the endpoint. Add to the agent's `ossec.conf` and restart the agent:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

### Test event
On PC1, run an encoded PowerShell command (it safely decodes to `whoami`, a classic recon command):

```powershell
powershell.exe -EncodedCommand dwBoAG8AYQBtAGkA
```

Wazuh raised alerts for:
- **Sysmon Event ID 1 (process creation):** `powershell.exe` spawned a PowerShell process running a base64-encoded command.
- **Sysmon Event ID 11 (file created):** an executable/script dropped in a folder commonly abused by malware (PowerShell writing a temp file to `AppData\Local\Temp`).

<img width="1697" height="266" alt="image" src="https://github.com/user-attachments/assets/69769bfa-2802-432c-bad7-ad79fec9b620" />

---

## Next Steps

- Generate a safe Suricata test alert with [testmynids.org](https://testmynids.org/) as a second confirmation, alongside the BlackSun test.
- Deploy Sysmon and its config to endpoints via GPO.
- Bring up Kali on External_WAN and run Hydra brute-force attempts against `Bob`, then correlate Suricata alerts (network layer), Event ID `4625` (failed logon), and Event ID `4740` (lockout) in Wazuh.
