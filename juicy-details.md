# Juicy Details

**Category:** Network / Log Forensics (Linux web server)
**Tools:** Linux log analysis (`access.log`, `auth.log`, `vsftpd.log`), grep/awk
**Skills demonstrated:** web-attack reconstruction, multi-source log correlation, SQL-injection and brute-force analysis, attacker tool identification



## Scenario

A Linux web server (an e-commerce app) was attacked. Three server logs are provided —
the web access log, the authentication log, and the FTP log. The task is to
reconstruct the full attack from the logs alone.



## Investigation

**1. Isolating the attacker.** All traffic in `access.log` originates from a single
IP — **192.168.10.5** — so the entire log is the attacker's activity. Status-code
summary showed the attack phases: many `401` (failed auth → brute force), many `500`
(malformed queries → SQL injection), and `200`/`201` (successful requests).

**2. Tools used (by occurrence in the log, via User-Agent strings):**

|Tool|Purpose|
|-|-|
|nmap|Port/service scanning (recon)|
|hydra|Brute-force authentication|
|sqlmap|Automated SQL injection|
|curl|Issuing crafted requests / retrieving dumped data|
|feroxbuster|Content discovery (finding hidden directories/files)|

**3. Brute force.** The attacker brute-forced the login endpoint
**`/rest/user/login`** with hydra. It succeeded — the successful login appears at
**11/Apr/2021:09:16:31 +0000**.

**4. SQL injection.** The endpoint **`/rest/products/search`** was vulnerable via the
**`q`** parameter. The attacker used a UNION-based injection to dump the users table:

```
GET /rest/products/search?q=qwert'))%20UNION%20SELECT%20id,email,password,...%20FROM%20Users--
→ 200 (via curl/7.74.0)
```

This retrieved **email and password** data from the `Users` table. The attacker also
scraped user **email addresses from the product reviews** section.

**5. Content discovery → hidden FTP endpoint.** feroxbuster brute-forced directory
paths and discovered the hidden **`/ftp`** endpoint. The attacker then requested two
backup files:

```
GET /ftp/coupons\_2013.md.bak
GET /ftp/www-data.bak
```

**6. FTP exfiltration.** Cross-referencing `vsftpd.log`: the attacker logged in to the
FTP service as **anonymous** and, about a minute after the GET requests, two
`OK DOWNLOAD` entries confirm both backup files were retrieved.

**7. Shell access.** Cross-referencing `auth.log`: the attacker gained remote shell
access over **SSH** as the user **`www-data`** (using credentials obtained earlier).



## Attack Chain

```
nmap scan (recon)
  → hydra brute-force on /rest/user/login  (success 11/Apr 09:16:31)
  → sqlmap SQL injection on /rest/products/search (param q)
      → UNION SELECT dumps email + password from Users (via curl)
      → emails also scraped from product reviews
  → feroxbuster content discovery → finds hidden /ftp
  → FTP (anonymous) → downloads coupons\_2013.md.bak, www-data.bak
  → SSH as www-data → shell access on the server
```



## Indicators of Compromise

|Type|Value|
|-|-|
|Attacker IP|192.168.10.5|
|Brute-forced endpoint|/rest/user/login|
|SQLi endpoint / parameter|/rest/products/search (param `q`)|
|Data stolen|user emails + passwords (Users table), review emails|
|Hidden endpoint found|/ftp|
|Files exfiltrated|coupons\_2013.md.bak, www-data.bak (via FTP, anonymous)|
|Shell access|SSH as www-data|
|Successful brute-force login|11/Apr/2021:09:16:31 +0000|



## Summary

A full web-application kill chain reconstructed from three correlated log sources:
recon (nmap) → credential brute-force (hydra) → SQL injection dumping user credentials
(sqlmap/curl) → content discovery of a hidden `/ftp` endpoint (feroxbuster) → file
exfiltration over anonymous FTP → SSH shell access as `www-data`.

