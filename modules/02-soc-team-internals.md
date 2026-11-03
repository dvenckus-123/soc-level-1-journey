# Module 2 — SOC Team Internals

**Path:** SOC Level 1 · **Status:** 🔄 in progress

Rooms in this module:

✅ SOC L1 Alert Triage <br>
✅ SOC L1 Alert Reporting <br>
✅ SOC Workbooks and Lookups <br>
✅ SOC Metrics and Objectives <br>

---

## Concepts, in my own words

_The main ideas from this module, explained the way I'd explain them to someone else._

**SOC L1 Alert Triage**  <br> <br>
[1] Prioritise alert (highest severity first) <br>
[2] Assign alert to yourself <br>
[3] Set status to "in progress" <br>
[4] Note alert name and description to understand the action described in the alert like suspicious login, malware, phishing, potential data exfiltration. <br>
[5] Note alert fields like host, IP or user to understand who is under threat like the affected user, hostname, network, website.<br>
[6] Review surrounding events, looking for suspicious actions shortly after or before the alert.<br>
[7] Does playbook exist for this category of alert? If yes -> follow steps in playbook. If no -> Investigate the activity in SIEM.<br>
[8] Make your final verdict on the alert (false positive/true positive) - Use threat intelligence platforms like Virus Total to verify your decision.<br>
[9] False positive -> add comment and close alert. True positive -> escalate to L2 for deeper analysis and remediation. <br><br>

**SOC L1 Alert Reporting**  <br> <br>
Use 5 W's when writing alert reports: <br><br>
Who? - Which user is affected? who clicked on a link, ran a command, downloaded a file? <br>
What? - What exact action was performed? e.g. - Phishing email impersonating Amazon.com / Phishing link blocked by firewall. <br>
When? - When did this suspicious activity start and end (timestamps). <br>
Where? - What device, IP or website was involved in the alert. e.g. - email and firewall datasources. <br>
Why? - The reasoning for your final verdict (false positive/true positive). Why it was suspicious. e.g. - Malicious URL and impersonation. <br><br>

Alert reporting, escalation and communication are essential skills for L1 analyst. <br><br>
Alert reporting -> Provides activity context for L2. <br>
Alert escalation -> Ensure threats are remediated in time. <br>
Alert communication -> Makes coordination between SOC and other departments clear and effective. <br><br>

**SOC Workbooks and Lookups**  <br><br>
Use identity inventory to get context about a user (Name, email, Role, Access) - e.g. - AD, Entra ID, BambooHR.  <br>
Use asset inventory to get context about host, IP, location, purpose. e.g. - AD, Entra ID, SIEM/EDR, MDM solution.  <br>
Use network diagrams to give meaning to IPs/ports you see in an alert.  <br>

  
Workbooks/playbooks streamline alert triage. <br><br>
Stage 1: Enrichment - Use identity inventory / asset inventory / threat intelligence to get information/context about user/system.  <br>
Stage 2: Investigation - Use gathered data/SIEM logs/other resources to make final verdict.  <br>
Stage 3: Escalation - Escalate to L2 or communicate issue to user/system owner.  <br><br>

**SOC Metrics and Objectives**  <br> <br>

Core Metrics: <br> <br>
Alert count (AC) = Total count of alerts received    (Measure overall load of the SOC analysts) <br>
False positive rate (FPR) = False positives / Total alerts   (Measures level of noise in the alerts) <br>
Alert escalation rate (AER) = Escalated alerts / Total alerts   (Measures experience of L1 analysts) <br>
Threat detection rate (TDR) = Detected threats / Total threats   (Measures reliability of the SOC team) 
<br><br>

Key performance indicators (Often shared with other parties as part of SLA): <br> <br>
MTTD (Mean Time To Detection) - Average time between the attack and it's detection by SOC tools. (Common SLA is 5min). <br>
MTTA (Mean Time To Acknowledgment) - Average time for L1 analyst to start triage of new alert. (Common SLA is 10min). <br>
MTTR (Mean Time To Response) - Average time it takes for SOC to stop the breach from spreading. (Common SLA is 60min). <br>
<br> <br>



## Tools & techniques

_Anything hands-on I used or practiced._

- Used Splunk to view and analyse email & firewall logs as part of phishing simulation exercise.

## What clicked / what I got stuck on

_Honest notes. The thing that confused me and how I worked it out is often the most useful bit to re-read later._

- Had to do the phishing simulation exercise a second time to better understand how to write a well structured report. Missed some important details during my first attempt: <br><br>

- Did not explain what the attacker was actually after (what the intent was). The malicious email asked the recipient to click the link and confirm their shipping information, so I should have stated that the goal was to harvest the personal or shipping details the user would enter on the linked page. <br>
- Did not check whether any other users received the same email from 'urgents@amazon.biz'. Searching the email logs in SIEM for that sender address is important to determine the scope of the campaign. <br>
- Remediation actions section should have included a recommendation to block the domain and IP to prevent future delivery attempts. Blocking the domain and IP is a critical step to prevent any other user in the organisation from reaching that phishing page. <br>
