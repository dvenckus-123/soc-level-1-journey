
## setup & output/screenshots

perform brute force with a word list containing 100 passwords then investigation with alert → pivot → timeline → conclusion.
* the test will use 100 passwords to brute force account lab.local\bob. passwords taken from rockyou 2009 data breach, which come with kali linux. `head -n 100 /usr/share/wordlists/rockyou.txt > /home/kali/password_list_100.txt`
* 

 brute force from terminal: `hydra -l bob -P /home/kali/password_list_100.txt rdp://172.16.10.2/lab.local`



Hydra output:
 <img width="1372" height="792" alt="image" src="https://github.com/user-attachments/assets/75179b34-dbc7-47b5-8151-85ded7f626e5" />

<br>

Suricata log (ET INFO RDP - Response To External Host (sid 2001330)):
<img width="1732" height="97" alt="image" src="https://github.com/user-attachments/assets/d4c51aaa-a0b1-48e6-b9b8-18fcf941a46f" />
Suricata plain-text signature from the Emerging Threats (emerging-info.rules) ruleset. (ET INFO RDP - Response To External Host (sid 2001330)):
<img width="1045" height="364" alt="image" src="https://github.com/user-attachments/assets/78ebe2b1-f494-4f95-bca5-ade337c13079" />
Rule text:
alert tcp $HOME_NET 3389 -> $EXTERNAL_NET any (msg:"ET INFO RDP - Response To External Host"; flow:established,to_client; content:"|03|"; offset:0; depth:1; content:"|D0|"; offset:5; depth:1; classtype:misc-activity; sid:2001330; rev:10; metadata:attack_target Client_and_Server, created_at 2010_07_30, deployment Perimeter, performance_impact Significant, confidence Medium, signature_severity Informational, updated_at 2023_04_25, reviewed_at 2024_05_02; target:src_ip;)

<br>

Suricata log (ET SCAN Behavioral Unusually fast Terminal Server Traffic (sid 2001972)):
<img width="1733" height="125" alt="image" src="https://github.com/user-attachments/assets/5ad55a1f-81f7-4366-a28e-39d173134e22" />
Suricata plain-text signature from the Emerging Threats (emerging-scan.rules) ruleset. (ET SCAN Behavioral Unusually fast Terminal Server Traffic (sid 2001972)):
<img width="1051" height="370" alt="image" src="https://github.com/user-attachments/assets/1b4caf4d-52f8-4985-99b7-196e53992391" />
Rule text:
alert tcp $EXTERNAL_NET any -> $HOME_NET 3389 (msg:"ET SCAN Behavioral Unusually fast Terminal Server Traffic Potential Scan or Infection (Inbound)"; flow:not_established,to_server; flags: S,12; threshold: type both, track by_src, count 20, seconds 360; classtype:network-scan; sid:2001972; rev:21; metadata:created_at 2010_07_30, confidence Medium, signature_severity Informational, updated_at 2023_11_14;)




Triggered custom 100100 rule from Wazuh in local_rules.xml:
<img width="1625" height="346" alt="image" src="https://github.com/user-attachments/assets/282f2493-e6d6-45d1-a19e-d4f64304f129" />


Wazuh alerts page with 100100 rule highlighted in red:
<img width="2523" height="912" alt="image" src="https://github.com/user-attachments/assets/f4284e06-a108-4da5-b50f-b78cce345e3b" />
<img width="2520" height="830" alt="image" src="https://github.com/user-attachments/assets/61e0023d-33d4-4bf2-9b77-fcb7f330f017" />
<img width="2523" height="831" alt="image" src="https://github.com/user-attachments/assets/1cedc1de-1822-412b-b71c-c0da3479336a" />
<img width="2520" height="826" alt="image" src="https://github.com/user-attachments/assets/be96aee4-7499-4928-a203-f203a8e8908c" />

Active directory Bob account locked confirmation:

<img width="311" height="406.5" alt="image" src="https://github.com/user-attachments/assets/5a8919c2-0196-47f8-8753-ae1e2cb7d839" />




# Detection Results: RDP Brute Force Against PC1

## Alert
The investigation starts from the alert a real analyst would actually triage first — not the noisiest one, the most actionable one. Several Suricata signatures fired at level 7 (informational), but the alert that matters is the one purpose-built for this exact scenario:

> **Rule 100100** — *"Possible brute-force attack: 5 failed Windows logons from the same source IP within 60 seconds"* — **Level 10**, fired at `2026-11-26 15:28:59.136`, **MITRE T1110 (Brute Force)**, tactic **Credential Access**.

This is the custom correlation rule built earlier in this project, triggered via `if_matched_sid` once rule `60122` (*"Logon Failure - Unknown user or bad password"*) matched five times from the same source within the 60-second window. This is the headline alert — everything else in this section is pivoting outward from it to build context.

---

## Pivot

### Pivot 1 — Individual Logon Failures
The individual logon failures underneath the correlation. The five **Event ID 4625** entries that fed rule `100100` all point at the same actor and method:
- **Target account:** `bob` in domain `lab.local`
- **Source IP:** `172.16.10.10` (kali linux attack machine)
- **Logon type:** `3` (network logon), authentication package `NTLM`
- **Timing:** Five failures landed in roughly **2.6 seconds** (`15:28:55.785` → `15:28:58.374`), not spread across the full 60-second window

### Pivot 2 — Network Layer Corroboration
The network layer, for independent corroboration. Two separate Suricata signatures fired, and they're telling two different stories worth distinguishing:
- **`ET INFO RDP - Response To External Host` (SID `2001330`)** fired **55 times**. Per its own rule text (`target:src_ip`, `flow:established,to_client`), this is actually flagging PC1's own RDP responses going out to an external host — an expected side effect of the port forward being hit, not a detection of the attack itself. Useful as volume context, not as a primary finding. (What it does: This rule fires every single time your internal machine (PC1) replies to an external IP over the RDP port.)

<br>

- **`ET SCAN Behavioral Unusually fast Terminal Server Traffic` (SID `2001972`)** fired **once**, at `15:29:00.855` — a genuine behavioral detection, tripped by its own threshold (20 SYNs from one source within 360 seconds). This is the real network-layer confirmation of brute-force-style connection behavior, independent of and corroborating the host-based Wazuh correlation.

### Pivot 3 — Attack Efficacy & Discrepancy Analysis
Checking whether the attack actually worked. This is the discrepancy worth flagging directly: Hydra's own output shows only **3 password attempts** (`password`, `princess`, `12345678`) before it aborted with `all children were disabled due too many connection errors`, reporting `0 of 1 target completed, 0 valid password found`. 

Hydra's RDP module is explicitly marked experimental, and FreeRDP repeatedly failed to establish the connection. Yet Windows logged **5 distinct failed logon events** in that same window. The counts don't match 1:1 — RDP's handshake behavior under Hydra clearly generated more Windows-side authentication attempts than Hydra's own password counter reflects. Worth stating plainly rather than papering over: **the attack technically failed and never got a real foothold, but it still generated enough failed logons to trip both detections.**

### Pivot 4 — Automated Containment
Did anything actually respond to this? Checking Active Directory after the fact: `bob`'s account was **locked out**. The account lockout policy configured earlier in this project triggered automatically, with no manual analyst action required.

---

## Timeline

| Time (UTC) | Source | Event |
| :--- | :--- | :--- |
| `15:28:45` | Kali / Hydra | Attack launched: `hydra -l bob -P password_list_100.txt rdp://172.16.10.2/lab.local` |
| `15:28:55.785` | PC1 / Event 4625 | Failed logon #1 — `bob`, NTLM, source `172.16.10.10` |
| `15:28:55.798` | PC1 / Event 4625 | Failed logon #2 |
| `~15:28:55.831` onward | pfSense / Suricata | First of many `ET INFO RDP - Response To External Host` alerts begin logging |
| `15:28:56.075` | PC1 / Event 4625 | Failed logon #3 |
| `15:28:58.098` | PC1 / Event 4625 | Failed logon #4 |
| `15:28:58.374` | PC1 / Event 4625 | Failed logon #5 — fifth match within the window, satisfies rule `100100`'s frequency condition |
| `15:28:59.136` | Wazuh | **Rule 100100 fires:** possible brute-force attack detected |
| `15:29:00.855` | pfSense / Suricata | **Behavioral scan threshold fires (SID `2001972`)** — network-layer corroboration, ~1.7s after the host-based alert |
| `15:29:02` | Kali / Hydra | Attack aborts: `0 of 1 target completed, 0 valid password found` |
| *(post-attack)* | Active Directory | `bob` account found locked out — automated containment confirmed |


---

## Conclusion

- **Verdict:** **True Positive.** A real, externally-sourced credential-guessing attempt against PC1's RDP service, correctly identified by a purpose-built correlation rule, corroborated independently at the network layer, and automatically contained by an account lockout policy before any analyst had to intervene.
- **MITRE ATT&CK:** **T1110 (Brute Force)**, specifically **T1110.001 (Password Guessing)** — multiple passwords attempted against a single known username.

### Lab Validation
- **What this confirms about the lab:** The full detection chain works end to end — perimeter (port forward + Suricata), host (Windows auditing via the Wazuh agent), SIEM correlation (the custom rule), and automated response (AD lockout) — without manual intervention at any layer.

<br><br>

---

## 13. Attack Simulation: RDP Brute Force Against Bob

**Goal:** run a real brute-force attempt against the domain account `bob` from Kali, then investigate it the way an analyst would: alert, pivot, timeline, conclusion.

### Setup

Wordlist: the first 100 passwords from the RockYou breach list, which ships with Kali.

```bash
head -n 100 /usr/share/wordlists/rockyou.txt > /home/kali/password_list_100.txt
```

Attack command, run from Kali against the pfSense WAN IP (the RDP port forward from section 11):

```bash
hydra -l bob -P /home/kali/password_list_100.txt rdp://172.16.10.2/lab.local
```

<img width="1372" height="792" alt="image" src="https://github.com/user-attachments/assets/75179b34-dbc7-47b5-8151-85ded7f626e5" />

> **Note on Hydra's output:** Hydra reported connection errors and `0 valid password found`, and its RDP module is marked experimental, so at first glance it looks like the attack never landed. But Windows logged 5 real failed logon events in that same window, and Bob's account ended up locked. The lockout only happens after real authentication attempts reach the domain controller, so the attempts were genuinely processed by Windows even though Hydra's own client-side view didn't reflect it cleanly. This gap between what the attacker tool reports and what actually happened on the target is worth knowing about going into the investigation below.

---

### Investigation

#### Alert

Several Suricata signatures fired, but the alert that matters is the one built for this exact scenario:

> **Rule 100100** — *"Possible brute-force attack: 5 failed Windows logons from the same source IP within 60 seconds"* — **Level 10**, fired at `2026-11-26 15:28:59.136`, **MITRE T1110 (Brute Force)**, tactic **Credential Access**.

This is the correlation rule built in section 12. It fired once rule `60122` ("Logon Failure – Unknown user or bad password") matched five times from the same source IP inside the 60-second window.

<img width="1625" height="346" alt="image" src="https://github.com/user-attachments/assets/282f2493-e6d6-45d1-a19e-d4f64304f129" />

#### Pivot 1: the individual logon failures

The five Event ID 4625 entries behind the correlation all point at the same actor and method:

- **Target account:** `bob`, domain `lab.local`
- **Source IP:** `172.16.10.10` (Kali)
- **Logon type:** `3` (network logon), package `NTLM`
- **Timing:** all five failures landed within about 2.6 seconds (`15:28:55.785` → `15:28:58.374`), well inside the 60-second window

<img width="2523" height="912" alt="image" src="https://github.com/user-attachments/assets/f4284e06-a108-4da5-b50f-b78cce345e3b" />

> 📸 **TRIM:** you have four near-identical screenshots of the Wazuh alerts page. Keep the clearest one or two (e.g. this one plus the one with 100100 highlighted) and drop the rest, unless each shows something different worth calling out.

#### Pivot 2: the network layer

Two Suricata signatures fired, and they're telling different stories:

- **`ET INFO RDP - Response To External Host` (SID 2001330)** fired 55 times. Its own rule logic (`target:src_ip`, response flowing `to_client`) shows this is just PC1's RDP service replying to an external host — a normal side effect of the port forward being hit, not a detection of the brute force itself. It's useful as volume context, not as a finding on its own.

  <img width="1732" height="97" alt="image" src="https://github.com/user-attachments/assets/d4c51aaa-a0b1-48e6-b9b8-18fcf941a46f" />

  ```
  alert tcp $HOME_NET 3389 -> $EXTERNAL_NET any (msg:"ET INFO RDP - Response To External Host"; flow:established,to_client; content:"|03|"; offset:0; depth:1; content:"|D0|"; offset:5; depth:1; classtype:misc-activity; sid:2001330; rev:10; metadata:attack_target Client_and_Server, created_at 2010_07_30, deployment Perimeter, performance_impact Significant, confidence Medium, signature_severity Informational, updated_at 2023_04_25, reviewed_at 2024_05_02; target:src_ip;)
  ```

- **`ET SCAN Behavioral Unusually fast Terminal Server Traffic` (SID 2001972)** fired once, at `15:29:00.855`. This one tripped its own threshold (20 SYN packets from one source within 360 seconds), which is a genuine behavioral detection of brute-force-style connection activity. It confirms the attack at the network layer, independently of the Wazuh host-based alert.

  <img width="1733" height="125" alt="image" src="https://github.com/user-attachments/assets/5ad55a1f-81f7-4366-a28e-39d173134e22" />

  ```
  alert tcp $EXTERNAL_NET any -> $HOME_NET 3389 (msg:"ET SCAN Behavioral Unusually fast Terminal Server Traffic Potential Scan or Infection (Inbound)"; flow:not_established,to_server; flags: S,12; threshold: type both, track by_src, count 20, seconds 360; classtype:network-scan; sid:2001972; rev:21; metadata:created_at 2010_07_30, confidence Medium, signature_severity Informational, updated_at 2023_11_14;)
  ```

#### Pivot 3: automated containment

Checking Active Directory after the fact: Bob's account was locked out. The lockout policy from section 10 triggered automatically, with no manual action needed.

<img width="311" height="406.5" alt="image" src="https://github.com/user-attachments/assets/5a8919c2-0196-47f8-8753-ae1e2cb7d839" />

---

### Timeline

| Time (UTC) | Source | Event |
| :--- | :--- | :--- |
| `15:28:45` | Kali / Hydra | Attack launched: `hydra -l bob -P password_list_100.txt rdp://172.16.10.2/lab.local` |
| `15:28:55.785` | PC1 / Event 4625 | Failed logon #1 — `bob`, NTLM, source `172.16.10.10` |
| `15:28:55.798` | PC1 / Event 4625 | Failed logon #2 |
| `~15:28:55.831` onward | pfSense / Suricata | First `ET INFO RDP - Response To External Host` alerts begin logging |
| `15:28:56.075` | PC1 / Event 4625 | Failed logon #3 |
| `15:28:58.098` | PC1 / Event 4625 | Failed logon #4 |
| `15:28:58.374` | PC1 / Event 4625 | Failed logon #5 — satisfies rule 100100's frequency condition |
| `15:28:59.136` | Wazuh | **Rule 100100 fires:** possible brute-force attack detected |
| `15:29:00.855` | pfSense / Suricata | Behavioral scan threshold fires (SID 2001972), ~1.7s after the host-based alert |
| `15:29:02` | Kali / Hydra | Hydra exits: `0 of 1 target completed, 0 valid password found` |
| *(post-attack)* | Active Directory | `bob` found locked out — automated containment confirmed |

---

### Conclusion

- **Verdict:** True Positive. A real, externally-sourced brute-force attempt against PC1's RDP service, correctly flagged by the custom correlation rule, confirmed independently at the network layer, and automatically contained by the account lockout policy before any analyst had to step in.
- **MITRE ATT&CK:** T1110 (Brute Force), specifically T1110.001 (Password Guessing) — multiple passwords tried against one known username.
- **On Hydra's own output:** it reported no valid password found and connection errors, but that's a limitation of Hydra's experimental RDP module reporting back to the attacker, not a sign the attack had no effect. The failed logons and the resulting lockout show the attempts genuinely reached Windows.

**What this confirms about the lab:** the full detection chain works end to end — perimeter (port forward + Suricata), host (Windows auditing via the Wazuh agent), SIEM correlation (the custom rule), and automated response (AD lockout) — with no manual intervention at any layer.
