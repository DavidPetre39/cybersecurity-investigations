# Boogeyman 2

**Category:** Memory Forensics
**Tools:** Volatility 3 (pslist, cmdline, netscan), olevba
**Skills demonstrated:** memory-dump analysis, process-chain reconstruction, Office macro analysis, payload-disguise identification

\---

## Scenario

Maxine, a Human Resources Specialist at Quick Logistics LLC, received a job
application with an attached resume. The resume was malicious and compromised her
workstation. The security team flagged suspicious commands, prompting the
investigation. **Only a memory dump was available.**

\---

## Attack Chain

**1. Approach.** The memory dump was analyzed with **Volatility 3**, extracting
process command lines (`cmdline`), running processes (`pslist`), and network
connections (`netscan`).

**2. Delivery + execution.** Opening the malicious resume launched `winword.exe`.
From `pslist`, `winword.exe` spawned `wscript.exe` (PID 4260), which executed the
JavaScript payload `update.js`.

**3. Macro analysis.** Using **olevba** to inspect the document's VBA macros revealed
that `update.png` (JavaScript disguised with a `.png` extension) was downloaded from a
malicious website when the document was opened, then saved as `update.js`.

**4. Second-stage binary.** `update.js` downloaded a second binary, `updater.exe`
(PID 6216).

**5. C2.** From `netscan`, `updater.exe` (PID 6216) established a C2 connection to
`128.199.95.189:8080`.

**6. Persistence.** The attacker established persistence via a `schtasks` command
(scheduled task).

\---

## Indicators of Compromise

|Type|Value|
|-|-|
|Stage 2 payload|update.png → update.js (C:\\ProgramData\\update.js)|
|Script engine|wscript.exe (PID 4260)|
|Second-stage binary|updater.exe (PID 6216)|
|C2|128.199.95.189:8080|
|Persistence|scheduled task (schtasks)|

\---

## Summary

A malicious resume opened in Word spawned `wscript.exe` to run a disguised JavaScript
payload (`update.png` → `update.js`), which downloaded `updater.exe` and established
C2, with persistence via a scheduled task — reconstructed entirely from a memory dump.

