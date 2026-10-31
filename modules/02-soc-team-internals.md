# Module 2 — SOC Team Internals

**Path:** SOC Level 1 · **Status:** 🔄 in progress

Rooms in this module:

- ✅ SOC L1 Alert Triage
- ⬜ SOC L1 Alert Reporting
- ⬜ SOC Workbooks and Lookups
- ⬜ SOC Metrics and Objectives

---

## Concepts, in my own words

_The main ideas from this module, explained the way I'd explain them to someone else._

- SOC L1 Alert Triage - <br> <br>
[1] &nbsp;Prioritise alert (highest severity first) <br>
[2] &nbsp;Assign alert to yourself <br>
[3] &nbsp;Set status to "in progress" <br>
[4] &nbsp;Note alert name and description to understand the action described in the alert like suspicious login, malware, phishing, potential data exfiltration. <br>
[5] &nbsp;Note alert fields like host, IP or user to understand who is under threat like the affected user, hostname, network, website.<br>
[6] &nbsp;Review surrounding events, looking for suspicious actions shortly after or before the alert.<br>
[7] &nbsp;Does playbook exist for this category of alert? If yes -> follow steps in playbook. If no -> Investigate the activity in SIEM.<br>
[8] &nbsp;Make your final verdict on the alert (false positive/true positive) - Use threat intelligence platforms like Virus Total to verify your decision.<br>
[9] &nbsp;False positive -> add comment and close alert. True positive -> escalate to L2 for deeper analysis and remediation. <br><br>

- SOC L1 Alert Reporting - <br> 


Use 5 W's when escalating alerts to L2 for further review/analysis/remediation: <br><br>
-Who? - Which user logs in, runs a command, downloads a file. <br>
-What? - What exact action was performed.<br>
-When? - When did this suspicious activity start and end.<br>
-Where? - What device, IP or website was involved in the alert.<br>
-Why? - The reasoning for your final verdict (false positive/true positive).<br><br>

Alert reporting, escalation and communication are essential skills for L1 analyst. <br><br>
-Alert reporting -> Provides activity context for L2. <br>
-Alert escalation -> Ensure threats are remediated in time. <br>
-Alert communication -> Makes coordination between SOC and other departments clear and effective. <br><br>

- SOC Workbooks and Lookups -

- SOC Metrics and Objectives -


## Tools & techniques

_Anything hands-on I used or practiced._

- 

## What clicked / what I got stuck on

_Honest notes. The thing that confused me and how I worked it out is often the most useful bit to re-read later._

-
