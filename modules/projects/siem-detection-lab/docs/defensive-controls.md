**Network Segmentation (External_WAN / Secure_LAN)**
The lab is split into an untrusted external zone and an isolated internal zone using separate virtual switches, with pfSense as the only path between them. This mirrors real enterprise network design, where corporate systems are never directly internet-facing, and forces every attack to cross a defined chokepoint rather than reach targets directly.

**Perimeter Firewall (pfSense)**
pfSense sits at the boundary between External_WAN and Secure_LAN, filtering and routing all traffic that crosses it. It’s the first layer any external attack has to pass, and the only thing standing between Kali and the internal network besides the deliberate port forward opened for the simulation.

**Intrusion Detection/Prevention (Suricata)**
Suricata runs on pfSense, inspecting traffic for known-malicious signatures using the ET Open ruleset. Running in IDS (alert-only) mode for this lab, it generates alerts on suspicious traffic like brute-force attempts without blocking them outright, which lets the attack reach Windows 11 so the full detection chain can be observed end to end.

**SIEM / Log Correlation (Wazuh)**
Wazuh acts as the central collection point for every log source in the lab — firewall, IDS, endpoint, file integrity, and threat intel enrichment all land here. It’s what turns scattered logs across five separate machines into a single place an analyst can actually triage from.

**Endpoint Visibility (Sysmon)**
Sysmon runs on Windows 11, logging detailed process creation, network connection, and system activity beyond what Windows logs natively. Using the SwiftOnSecurity community config rather than Sysmon’s bare defaults keeps the signal high and the noise low, which matters because raw, untuned Sysmon output is close to unusable at scale.

**File Integrity Monitoring (FIM)**
Wazuh’s FIM module watches a defined directory on Windows 11 in real time, alerting the moment a file is added, modified, or deleted. It’s the control that turns an unexpected file drop — the kind of thing that happens during a real compromise — into something a SOC can actually see and act on.

**Threat Intelligence Enrichment (VirusTotal)**
When FIM detects a new or changed file, Wazuh automatically submits its hash to VirusTotal and attaches the reputation result to the alert. This chains two controls together — detection, then enrichment — so an analyst gets a verdict on whether a file is known-malicious without manually checking it themselves, speeding up triage.

**Active Directory Account Lockout Policy**
A domain lockout threshold stops repeated failed logons after a set number of attempts, turning a sustained brute-force attempt into an automatic lockout event rather than an endless string of individual failures. It also means the brute-force simulation generates domain-account authentication events instead of local-account ones, which is the far more realistic scenario for a SOC analyst to actually encounter and investigate.
