# Cybersecurity Investigations

Self-directed DFIR and SOC investigations across network, memory, and endpoint
forensics. Each write-up reconstructs the full attack chain from the available
evidence, not just a list of indicators, but how the attacker gained access,
what techniques they used, and how each artifact connects to the next.

---

## Investigations

| Investigation | Category | Key Techniques |
|---------------|----------|----------------|
| [Tempest](tempest.md) | Endpoint / Memory | Follina (CVE-2022-30190), msdt abuse, C2, scheduled-task persistence |
| [Boogeyman 1](boogeyman-1.md) | Endpoint / PowerShell | Phishing → .lnk, Seatbelt enumeration, hex-encoded DNS exfiltration |
| [Boogeyman 2](boogeyman-2.md) | Memory Forensics | Volatility, macro → JScript → payload chain, Outlook attachment analysis |
| [Boogeyman 3](boogeyman-3.md) | Active Directory | ISO/HTA delivery, UAC bypass, pass-the-hash, lateral movement, DCSync |
| [OpenWire](openwire.md) | Network Forensics | Apache ActiveMQ RCE (CVE-2023-46604), Java deserialization, ProcessBuilder, Meterpreter/Mettle C2 |
| [Disgruntled](disgruntled.md) | Endpoint Forensics (Linux) | Insider threat, privilege abuse, rogue sudo account, cron logic bomb |
| [Juicy Details](juicy-details.md) | Log Forensics (Linux web) | Web attack kill chain — nmap, hydra, sqlmap SQLi, feroxbuster, FTP/SSH exfil, multi-source log correlation |
| [HawkEye](hawkeye.md) | Network Forensics | AD/Kerberos enumeration, SYSVOL/GPP hunting, HawkEye Reborn v9 infostealer, SMTP exfiltration |


---

## Approach

Every investigation follows the same method:

1. **Establish the baseline** — what normal looks like, so anomalies stand out.
2. **Identify the indicators** — malicious IPs, domains, hashes, processes, files.
3. **Build the chain** — link each indicator to the next by cause and effect:
   what spawned this process, what did it download, what did that execute.
4. **Reconstruct the timeline** — order the events into a single narrative.
5. **Document** — the attack story, the IOCs, and one clear takeaway.

The goal is always the full picture: not "here is a suspicious file," but
"this file was dropped by X, executed by Y, and established C2 to Z."

---

## Tools

| Area | Tools |
|------|-------|
| Network analysis | Wireshark, NetworkMiner |
| Memory forensics | Volatility 3 |
| Endpoint / disk | FTK Imager, Registry Explorer, Sysmon/Event Log analysis |
| Log analysis & SIEM | Splunk (SPL), Elastic / Kibana (KQL) |
| Decoding & triage | CyberChef, strings, VirusTotal |
| Lab | Isolated REMnux VM for malware handling |

---

## Background

Economic Informatics student focused on blue-team security — SOC analysis,
digital forensics, and incident response. These investigations are part of a
self-directed path toward detection engineering, building both the ability to
reconstruct attacks and, increasingly, to write the detections that catch them.
