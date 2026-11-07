# Boogeyman 3

**Category:** Active Directory / Endpoint Forensics
**Tools:** Elastic / Kibana (KQL), process \& network event analysis
**Skills demonstrated:** AD attack-path reconstruction, lateral movement analysis, pass-the-hash and DCSync identification, evidence-driven correlation



## Scenario

After an initial email compromise, the Boogeyman threat actors targeted the CEO, Evan
Hutchinson. Evan opened a questionable attachment, saw nothing happen, and reported
the phishing email to the security team. The investigation traces a full Active
Directory compromise across multiple machines.



## Attack Chain

**1. Delivery + execution.** An **ISO disguised as a PDF** mounted as a drive
containing an **HTA** file, executed by `mshta.exe` (PID 6392). From its children,
`xcopy` and `rundll32` both operated on `review.dat`.

**2. Implant.** The HTA used `xcopy` to copy `review.dat` — a **disguised DLL** — from
the ISO to the user's Temp folder, then executed it via
`rundll32.exe review.dat,DllRegisterServer`, establishing C2 to `165.232.170.151:80`
and persistence via a scheduled task named "Review".

**3. UAC bypass.** The attacker used **fodhelper.exe** (an auto-elevating Windows
binary) to bypass UAC and gain full administrative privileges.

**4. Credential dumping.** **mimikatz** (downloaded from GitHub) dumped credentials
from memory, yielding the NTLM hash of `itadmin`. Using `sekurlsa::pth`, the attacker
performed a **pass-the-hash** to spawn a process with itadmin's privileges.

**5. Credential discovery + lateral movement.** Enumerating file shares surfaced
`IT\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\_Automation.ps1` on a remote share, containing hardcoded credentials for
`QUICKLOGISTICS\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\allan.smith` (`Tr!ckyP@ssw0rd987`). Using these, the attacker moved
laterally to **WKSTN-1327** via `Invoke-Command` over **WinRM** — confirmed by the
parent process of the malicious command on WKSTN-1327 being `wsmprovhost.exe`, the
WinRM service host.

**6. Domain compromise (DCSync).** On WKSTN-1327, mimikatz dumped the `administrator`
NTLM hash (`00f80f2538...`). With domain-admin access, the attacker reached the
**Domain Controller** and performed a **DCSync** attack — abusing the legitimate
domain-replication feature to impersonate a DC and request password hashes directly.
Besides `administrator`, the account `backupda` (a backup domain admin) was dumped.

**7. Impact.** With the domain fully compromised, a ransomware binary was downloaded
from `http://ff.sillytechninja.io/ransomboogey.exe` to detonate across the
environment.



## Indicators of Compromise

|Type|Value|
|-|-|
|Delivery|ISO (disguised as PDF) → HTA (mshta.exe, PID 6392)|
|Implant|review.dat (disguised DLL), run via rundll32 DllRegisterServer|
|C2|165.232.170.151:80|
|Persistence|scheduled task "Review"|
|UAC bypass|fodhelper.exe|
|Cred-dump tool|mimikatz (GitHub)|
|Compromised accounts|itadmin, allan.smith, administrator, backupda|
|Hardcoded creds source|IT\_Automation.ps1 (remote share)|
|Lateral movement|Invoke-Command / WinRM → WKSTN-1327 (wsmprovhost.exe)|
|Ransomware|http://ff.sillytechninja.io/ransomboogey.exe|



## Summary

A full Active Directory compromise: one phishing email led to a single host, then
privilege escalation (fodhelper UAC bypass), credential theft (mimikatz,
pass-the-hash), lateral movement over WinRM using credentials hardcoded in a script,
domain-controller compromise via DCSync, and finally ransomware deployment.

