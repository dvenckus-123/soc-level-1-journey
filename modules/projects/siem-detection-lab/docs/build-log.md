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

<img width="452.25" height="533.25" alt="image" src="https://github.com/user-attachments/assets/1c9abf76-d0b9-4bd7-a233-6abcff191866" />

4. **Create an OU and a test user** (`Bob`).
5. **Join Windows 11 to the domain:** first set its primary DNS to the DC (`192.168.10.10`), because it has to find the DC before it can join. Then `System > Domain`, enter the domain name and reboot.


### Account lockout policy (GPO)
1. Open `gpmc.msc`, right-click **Default Domain Policy** and choose Edit.
2. Go to `Computer Configuration > Policies > Windows Settings > Security Settings > Account Policies > Account Lockout Policy`.
3. Set **Account lockout threshold** to `5`.
4. Run `gpupdate /force` on the DC.
5. **Test:** enter a wrong password 5 times for Bob on Windows 11, then check the DC's Security log for Event ID `4740` (account lockout).

<img width="1444" height="931" alt="image" src="https://github.com/user-attachments/assets/f24c9056-cbdd-433c-823f-3b19c20d70eb" />


---


## 11. Log Correlation in Wazuh: Brute-Force Detection Rule

**Goal:** turn repeated failed logons into a single alert that says "this looks like a brute-force attack".

**Problem:** on their own, five failed logons produce five separate low-severity "logon failed" alerts, and nothing that tells an analyst they are related. A **correlation rule** links those events together, which turns raw telemetry into the signal a SOC analyst actually looks for.

### How correlation works in Wazuh

Wazuh implements correlation through [rules](https://documentation.wazuh.com/current/user-manual/ruleset/ruleset-xml-syntax/rules.html) that relate multiple events over time. The options used here:

| Option | Purpose |
| :--- | :--- |
| `if_matched_sid` | Builds on alerts already generated by an existing (base) rule |
| `frequency` | How many matching events are required |
| `timeframe` | The window (in seconds) in which those events must occur |
| `same_field` | Requires a decoded field to be identical across the events |

### Step 1: Identify the base rule

A correlation rule needs the rule ID that detects a *single* failed login. The ID is in the `rule.id` field of the alert in the Wazuh dashboard.

I confirmed it by deliberately failing a logon on the Windows 11 machine and checking the dashboard:
- Wazuh rule **`60122`** fired.
- It matches the Windows event **4625** ("An account failed to log on"), visible in Wazuh as `data.win.system.eventID: 4625`.

<img width="1819" height="244" alt="image" src="https://github.com/user-attachments/assets/8f836c60-4087-416f-b48c-4799158835fc" />

### Step 2: Write the correlation rule

Add this to `/var/ossec/etc/rules/local_rules.xml` on the Wazuh manager (the same file as the Suricata rule, with its own unique ID in the custom range):

```xml
<group name="custom,authentication,">

  <rule id="100100" level="10" frequency="5" timeframe="60">
    <if_matched_sid>60122</if_matched_sid>
    <same_field>win.eventdata.ipAddress</same_field>
    <description>Possible brute-force attack: 5 failed Windows logons from the same source IP within 60 seconds</description>
    <mitre>
      <id>T1110</id>
    </mitre>
  </rule>

</group>
```

Then restart the manager to load it:

```bash
sudo systemctl restart wazuh-manager
```

**What each line does**
- `if_matched_sid`: builds on alerts from the original failed-login rule (`60122`).
- `frequency="5"`: requires five matching events.
- `timeframe="60"`: counts them within 60 seconds.
- `same_field`: requires all five events to come from the same source IP.
- `level="10"`: gives the correlated alert a much higher severity than the individual failures.
- `<mitre><id>T1110</id>`: tags the alert with the MITRE ATT&CK technique **Brute Force**, so it shows up under that technique in the dashboard.

> **Gotcha: `same_srcip` doesn't work for Windows events.**
> Windows events decoded through `eventchannel` don't populate Wazuh's standard `srcip` field. The source IP lives inside a dynamic field instead (`win.eventdata.ipAddress`), so the rule has to use `same_field` with that path rather than the usual `same_srcip`.

> ⚠️ **Check:** what value did `win.eventdata.ipAddress` hold in your test? For logons typed at the console, Windows often records `-` or a loopback address rather than a real IP, which means the rule is grouping on that placeholder. It still fires in this test, but it matters once the failed logons come over the network (the Hydra test from Kali, where it should hold the attacker's IP). Worth noting what you saw.

### Step 3: Validate the rule

Generate a real brute-force pattern: fail a Windows logon **5 times in under 60 seconds** with the wrong password. Then check the Wazuh dashboard for both:
- the five individual `60122` alerts, and
- the new correlated `100100` alert alongside them.

**Result:** rule `100100` (level 10) fired right after the 5th failed logon (event ID 4625), with `mitre.id` showing `T1110` (Brute Force).

<img width="1877" height="444" alt="image" src="https://github.com/user-attachments/assets/44ec3d52-be25-4e26-bdbb-085d8e6a52c3" />


---


## 12. Sysmon on Windows 11

Sysmon stays resident across reboots and logs detailed system activity to the Windows event log: process creation, network connections, file creation time changes, and more.

**Config choice:** raw Sysmon output is mostly noise, so I use the [SwiftOnSecurity `sysmonconfig-export.xml`](https://github.com/SwiftOnSecurity/sysmon-config) rulebook, which defines what to log and what to exclude.

### Install
1. Download [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) and extract it, plus the config file, to `C:\sysmon`.
2. From an admin prompt:
   ```powershell
   .\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
   ```
3. To update the config later: `.\Sysmon64.exe -c sysmonconfig-export.xml`


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
- Bring up Kali on External_WAN and run Hydra brute-force attempts against `Bob`, then correlate Suricata alerts (network layer), Event ID `4625` (failed logon), and Event ID `4740` (lockout) in Wazuh.

## 11. Kali Linux (Attacker Machine)

**Goal:** add an attacker on the untrusted External_WAN network, and expose RDP on PC1 so brute-force attempts arrive as real network logons with the attacker's IP.

### Install Kali

1. Download the pre-built VirtualBox image from the [official Kali page](https://www.kali.org/get-kali/#kali-virtual-machines).
2. Extract it:
   ```bash
   7z x kali-linux-2026.2-virtualbox-amd64.7z
   ```
3. Add the `.vbox` file to VirtualBox via `Machine > Add` ⚠️ (your original said "Open"; check the menu name in your version).
4. Enable adapter 1 and attach it to `NAT Network` > `External_WAN`.
5. Set a static IP inside the OS, per the addressing plan:
   - IP `172.16.10.10/24`
   - Gateway `172.16.10.1`
   - DNS `8.8.8.8`

### Expose RDP on PC1 through pfSense

Kali sits on the WAN side, so it can't reach `192.168.10.20` directly. I added a port forward on pfSense so RDP reaches PC1 through the firewall. This is deliberate: it gives the attacker something to attack, and makes the brute-force traffic cross the perimeter where Suricata can see it. (Don't do this on a real network.)

`Firewall > NAT > Port Forward`:

| Setting | Value |
| :--- | :--- |
| Interface | WAN |
| Protocol | TCP |
| Destination | WAN address |
| Destination port | `3389` |
| Redirect target IP | `192.168.10.20` (PC1) |
| Redirect target port | `3389` |

<img width="1431" height="307" alt="image" src="https://github.com/user-attachments/assets/05a878c0-c23a-45e1-ac6b-85a8a7528bf6" />

*(Port forward rule)*

<br>

pfSense also creates matching WAN firewall rule automatically to allow that traffic through:
<img width="1435" height="282" alt="image" src="https://github.com/user-attachments/assets/f2d08a4e-b466-4c3d-9783-0f3de7ecb57e" />
*(WAN rule)*


### Confirm PC1 is listening

Before testing from Kali, make sure PC1 is actually accepting RDP. It has to be enabled in Windows 11 under `Settings > System > Remote Desktop`. Then check from PC1:

```powershell
netstat -aon | findstr "3389"
```

<img width="1067" height="84" alt="image" src="https://github.com/user-attachments/assets/3c2b89a9-e65d-4872-b3b8-fe7e1276ba6a" />

### Test from Kali

Target the **pfSense WAN IP**, not PC1's internal address. The port forward lives on the WAN interface, and Kali has no route to `192.168.10.0/24`.

```bash
nmap -p 3389 172.16.10.2
```

**Result:** port `3389/tcp` is **open** and the service is `ms-wbt-server`, the Windows RDP protocol. The full path works: Kali, then pfSense WAN, then the port forward, then PC1.

<img width="822" height="281" alt="image" src="https://github.com/user-attachments/assets/f090a307-8041-47d8-8d19-a70a0c869b27" />

This also depends on the earlier **Block private networks** fix (section 4). Without it, pfSense would drop Kali's private-range traffic before the port forward ever applied.


Verify from kali that port forwarding and windows RDP are working correctly with `xfreerdp /v:172.16.10.2 /u:bob /cert:ignore`. enter domain lab.local and the password for bob's account. this will establish an interactive windows session.

> ✏️ ADD (gotcha) if the port showed as `closed` or `filtered` on the first attempt, what it was and which of the three layers was responsible: RDP disabled on PC1, the pfSense rule, or the Windows firewall.


