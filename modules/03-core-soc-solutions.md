# Module 3 — Core SOC Solutions

**Path:** SOC Level 1 · **Status:** 🔄 in progress

Rooms in this module:

✅ Introduction to EDR <br>
✅ Introduction to SIEM <br>
✅ Splunk: The Basics <br>
⬜ Introduction to SOAR <br>

---

## Concepts, in my own words

_The main ideas from this module, explained the way I'd explain them to someone else._

**-Introduction to EDR** <br><br>

- Advanced endpoint protection tool that provides Visibility, Detection & Response. <br>
- AV is signature-based. Detects threats by comparing them to a database of known malicious behaviours. EDR goes beyond that. It detects threats based on certain behaviours. If an endpoint deviates from normal baseline behaviour, EDR picks it up immediately and logs an alert.   <br>
- Constantly monitors endpoints using telemetry received from agents. <br>
- Some of the telemetry data it collects is: process executions and terminations, network connections, command line activity, file and folder modifications and registry modifications. <br>
- When a threat is detected, you can respond by isolating the host, killing the process, placing a malicious file into quarantine or establishing remote access shell to the endpoint to carry out additional work like executing scripts. <br>

**-Introduction to SIEM** <br><br>

* Collects logs from endpoints, servers, firewalls via agent/syslog protocol/API.<br>
↓
* They come in different formats so it needs to be cleaned up to have a consistent format. This is known as standardisation (or normalisation/parsing) <br>
↓
* Then correlation to form patterns/relationships between the logs as individual logs are not very useful. <br>
↓
* Then detect malicious activities using detection rules (e.g. - If log source is WinEventLog AND EventId is 104 - Trigger an alert: EVENT LOG CLEARED)

**-Splunk: The Basics** <br><br>

Splunk is a SIEM tool with 3 core components. Forwarder, Indexer and Search Head. <br>

* Forwarder: Forwarder is an agent that collects logs from endpoints and sends it to Indexer.

* Indexer: Collects & processes logs from Forwarder. Parses and normalises the data into field-value pairs and stores results as events. 
  * Parsing = Extracting values from raw, unstructured/semi-structured log text and turning them into individual, labeled fields such as src_ip="192.168.2.10".
  * Normalisation = aligning those extracted fields across different vendors or systems into a shared, common schema such as deciding that src_ip, client_ip and remote_addr from three different tools should be renamed to a single standard fied like source_ip.

* Search Head: Used to search indexed logs using SPL(Search Processing Language). When the user performs a search, it's sent to the indexer and the relevant events are returned as field-value pairs. 

<img width="582" height="178" alt="image" src="https://github.com/user-attachments/assets/985e0c0d-272d-4fbc-a314-4394d3aa8508" />

<img width="500" height="300" alt="image" src="https://github.com/user-attachments/assets/209cb1b3-d5c8-48bc-bf87-a0b8f6960223" />





**-Introduction to SOAR** <br><br>




## Tools & techniques

_Anything hands-on I used or practiced._

* Ingested vpn-logs.json into Splunk. Practiced SPL to search for relevant events like returning the username associated with IP 107.14.182.38, number of events originated from all countries except France and how many events were captured by user Maleena. 



## What clicked / what I got stuck on

_Honest notes. The thing that confused me and how I worked it out._

- 
