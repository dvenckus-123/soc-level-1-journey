# Module 1 — Blue Team Introduction

**Path:** SOC Level 1 · **Status:** 🔄 in progress

Rooms in this module:

- ✅ Junior Security Analyst Intro
- ✅ SOC Role in Blue Team
- ✅ Humans as Attack Vectors
- ✅ Systems as Attack Vectors

---

## Concepts, in my own words

_The main ideas from this module, explained the way I'd explain them to someone else._

- Junior Security Analyst - The main responsibility of a security analyst is to monitor the network for suspicious activity and stop attacks as soon as they are detected. SOC analysts work with other members of the SOC to keep everything secure. Some of those are Senior Analyst, SOC engineer, Incident responder and SOC manager. SIEM is the primary tool used by SOC analyst. It pulls logs from various sources in the network and presents them in an easy to read format. These logs are presented in the form of alerts. If a serious alert is detected, such as unauthorised log in from an unkown IP address, they would normally escalate this alert to senior analyst or the SOC manager for further review/investigation.

- SOC Role in Blue Team - The blue team looks after the cyber defence of a business. There is an internal SOC which is a dedicated department inside a company who look after a single company. There is also MSSP (Managed Security Service Provider) which is an outsourced security provider who may have many customers. The CISO, who has expert level knowledge about cyber security reports to the CEO and is in charge of all the major SOC department decisions. The incident response team or CIRT are called up when a major incident is logged and it requires deep investigation, analysis and resolution. There is also specialised roles such GRC analyst, who make sure the company is compliant with regulations.

- Humans as Attack Vectors - Humans are the weakest link in cyber security. Why spend days exploiting vulnerabilities in secure networks when you can just get someone to hand you the key? Phishing, the most common method of social engineering is the most common ways attackers get a foothold into the system. Phishing is tricking the user into handing them over sensitive information. Could be an email asking you to urgently click a malicious link or could be a phone call where an attacker impersonates someone from your company. Billions of phishing emails are sent daily, which is a shocking statistic. Anti-phishing tools are a must in every business. They would block most of these automatically. But still some of them might slip through, so our job as a SOC analyst is to detect and respond to those that got through.

- Systems as Attack Vectors - 81% of breaches happen through stolen or breached passwords. You can check if your password was found in a data breach by visiting https://haveibeenpwned.com/Passwords. It is extremely important to educate users and IT departments to use strong passwords and to never re-use passwords. Attackers can also get in through software vulnerabilities. As soon as a vulnerability is found, it is given a CVE number. The only response to a CVE is a patch. When a CVE is disclosed, the attackers rush to exploit the vulnerability and defenders rush to patch it. Sometimes the CVE is disclosed but a patch hasn't been made public yet by the vendors, so workarounds and mitigation techniques must be put in place until the patch is released. When a trusted app starts running malicious code, it usually points to a supply chain attack, which is one of the most destructive types of attacks. Your computer runs many apps such as chat apps, development, web, entertainment. These apps are never coded from scratch and use libraries of code that allow them to perform different functions. If that library of code is infected, and an update is pushed, this can infect hundreds or thousands of machines. 

## Tools & techniques

_Anything hands-on I used or practiced._

- 

## What clicked / what I got stuck on

_Honest notes. The thing that confused me and how I worked it out is often the most useful bit to re-read later._

-


