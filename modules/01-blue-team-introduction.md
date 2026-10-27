# Module 1 — Blue Team Introduction

**Path:** SOC Level 1 · **Status:** 🔄 in progress

Rooms in this module:

- ✅ Junior Security Analyst Intro
- ✅ SOC Role in Blue Team
- ✅ Humans as Attack Vectors
- [ ] Systems as Attack Vectors

---

## Concepts, in my own words

_The main ideas from this module, explained the way I'd explain them to someone else._

- Junior Security Analyst - The main responsibility of a security analyst is to monitor the network for suspicious activity and stop attacks as soon as they are detected. SOC analysts work with other members of the SOC to keep everything secure. Some of those are Senior Analyst, SOC engineer, Incident responder and SOC manager. SIEM is the primary tool used by SOC analyst. It pulls logs from various sources in the network and presents them in an easy to read format. These logs are presented in the form of alerts. If a serious alert is detected, such as unauthorised log in from an unkown IP address, they would normally escalate this alert to senior analyst or the SOC manager for further review/investigation.

- SOC Role in Blue Team - The blue team looks after the cyber defence of a business. There is an internal SOC which is a dedicated department inside a company who look after a single company. There is also MSSP (Managed Security Service Provider) which is an outsourced security provider who may have many customers. The CISO, who has expert level knowledge about cyber security reports to the CEO and is in charge of all the major SOC department decisions. The incident response team or CIRT are called up when a major incident is logged and it requires deep investigation, analysis and resolution. There is also specialised roles such GRC analyst, who make sure the company is compliant with regulations.

- Humans as Attack Vectors - Humans are the weakest link in cyber security. Why spend days exploiting vulnerabilities in secure networks when you can just get someone to hand you the key? Phishing, the most common method of social engineering is the most common ways attackers get a foothold into the system. Phishing is tricking the user into handing them over sensitive information. Could be an email asking you to urgently click a malicious link or could be a phone call where an attacker impersonates someone from your company. Billions of phishing emails are sent daily, which is a shocking statistic. Anti-phishing tools are a must in every business. They would block most of these automatically. But still some of them might slip through, so our job as a SOC analyst is to detect and respond to those that got through. 

## Tools & techniques

_Anything hands-on I used or practiced._

- Used a simple browser based SIEM tool which tried to mimic the real thing.
- Found 2 critical alerts which was an unauthorised log in attempt followed by log in successful alert.
- Copied the attackers IP into an IP look-up database to check if it was malicious.
- Once confirmed to be malicious, escalated the incident over to the SOC manager for further review/investigation.

## What clicked / what I got stuck on

_Honest notes. The thing that confused me and how I worked it out is often the most useful bit to re-read later._

-


