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

- SOC L1 Alert Triage -
  1\. Prioritise alert (highest severity first)
  2) Assign alert to yourself
  3) Set status to "in progress"
  4) Note alert name and description to understand the action described in the alert like suspicious login, malware, phishing, potential data exfiltration. 
  5) Note alert fields like host, IP or user to understand who is under threat like the affected user, hostname, network, website.
  6) Review surrounding events, looking for suspicious actions shortly after or before the alert.
  7) Does playbook exist for this category of alert? If yes -> follow steps in playbook. If no -> Investigate the activity in SIEM.
  8) Make your final verdict on the alert (false positive/true positive) - Use threat intelligence platforms like Virus Total to verify your decision.
  9) False positive -> add comment and close alert. True positive -> escalate to L2 for deeper analysis and remediation. 

- SOC L1 Alert Reporting -

- SOC Workbooks and Lookups -

- SOC Metrics and Objectives -


## Tools & techniques

_Anything hands-on I used or practiced._

- 

## What clicked / what I got stuck on

_Honest notes. The thing that confused me and how I worked it out is often the most useful bit to re-read later._

-
