# Boogeyman 1

**Category:** Endpoint / PowerShell Log Analysis
**Tools:** lnkparse, jq, grep, CyberChef
**Skills demonstrated:** phishing attachment analysis, PowerShell script-block log analysis, tool identification by behavior, DNS exfiltration reconstruction

\---

## Scenario

Julianne, a finance employee at Quick Logistics LLC, received a follow-up email about
an unpaid invoice from a business partner. The attached document was malicious and
compromised her workstation. The security team flagged the suspicious execution, and
other finance employees reported similar phishing — a targeted attack attributed to a
new threat group, "Boogeyman," known for targeting the logistics sector.

\---

## Attack Chain

**1. Delivery.** The phishing email carried a ZIP containing a `.lnk` file named
`Invoice\_20230103`. Parsing it with **lnkparse** revealed the command it executed on
the target.

**2. Log processing.** The PowerShell logs (JSON) were flattened into a timestamped
text file so they could be read as a chronological narrative with `grep` — turning
hundreds of script blocks into a readable timeline.

**3. C2.** The payload established a C2 connection to `cdn.bpakcaging.xyz` on port
8080, beaconing for commands.

**4. Enumeration.** The attacker downloaded `Seatbelt.ps1` — an enumeration tool that
collects local security-relevant information. (Identified by behavior: the attacker's
trial-and-error with the tool's distinctive `-group=` syntax.)

**5. Credential targeting.** The attacker located and accessed `protected\_data.kdbx`
(a KeePass password database) and downloaded `sq3.exe` (SQLite3) to read the Sticky
Notes database — likely hunting for the KeePass master password.

**6. Exfiltration.** The attacker exfiltrated the `.kdbx` by **hex-encoding** it and
sending it in chunks as **DNS queries via nslookup** to the attacker's server
(`167.71.211.113`) — data smuggled out inside DNS, bypassing firewall egress rules.

\---

## Indicators of Compromise

|Type|Value|
|-|-|
|Lure file|Invoice\_20230103.lnk (inside ZIP)|
|C2|cdn.bpakcaging.xyz:8080|
|Enumeration tool|Seatbelt.ps1|
|Database tooling|sq3.exe (SQLite3)|
|Target file|protected\_data.kdbx (KeePass vault)|
|Exfil method|hex-encoded DNS queries via nslookup|
|Exfil destination|167.71.211.113|

\---

## Summary

A phishing `.lnk` launched a PowerShell-based intrusion: C2 over HTTP, Seatbelt
enumeration, and theft of a KeePass vault exfiltrated via hex-encoded DNS queries.

