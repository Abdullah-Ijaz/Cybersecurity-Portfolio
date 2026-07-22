# Pickle Rick: TryHackMe

**Platform:** TryHackMe  **Difficulty:** Easy  **Category:** Web Exploitation / Linux Privilege Escalation

## Executive Summary & Methodology
This engagement demonstrates a principle that carries over from both incident response and clinical triage: individually minor findings, when chained, produce a critical outcome. No single weakness here was severe on its own: an exposed credential, an authenticated command interface, an overly permissive `sudo` policy. Stacked together, they produced full root-level compromise. The methodology below is deliberately linear: map the surface, establish a foothold, then escalate.

---

## Phase 1: Enumeration

### The Initial Scan
The assessment began with a service and default-script scan to define the attack surface.

```bash
nmap -sV -sC -oN initial_scan.txt <Target_IP>
```

**Critical Findings:**
* **Port 22 (SSH):** Standard OpenSSH, a potential pivot but useless without credentials.
* **Port 80 (HTTP):** Apache. The only interactive surface, and therefore the primary vector.

### Web Content Discovery
With a two-port host, enumeration focused entirely on the web application.

```bash
gobuster dir -u http://<Target_IP> -w /usr/share/wordlists/dirb/common.txt
```

Most paths returned `403 Forbidden`, but two were reachable and immediately relevant: `login.php` and `robots.txt`.

### Credential Harvesting
A complete credential pair was recovered from two client-visible sources:
* **Username:** left in an HTML comment in the homepage source (`View Source` / `Ctrl+U`): `<!-- Note to self, remember username! R1ckRul3s -->`
* **Password:** the sole entry in `robots.txt`:

```bash
curl http://<Target_IP>/robots.txt
# Wubbalubbadubdub
```

*Risk Analysis Note: Anything served to the client (HTML comments, `robots.txt`, JavaScript) is readable by anyone. `robots.txt` is a crawler directive, never an access control. Treating obscurity as security is a recurring root cause in real assessments.*

---

## Phase 2: Foothold via the Command Panel
Authenticating to `login.php` with `R1ckRul3s : Wubbalubbadubdub` presented a "Command Panel", effectively a web shell that executes attacker input as OS commands. `whoami` and `ls -a` confirmed code execution as the `www-data` service account and revealed the first ingredient file.

### Error & Correction 1: The `cat` Filter
* **The Action:** `cat Sup3rS3cretPickl3Ingred.txt`
* **The Error:** The command was rejected: the panel filters `cat`.
* **The Correction:** The filter is a naive blocklist, and dozens of utilities read files. Switching to `less` bypassed it entirely:

```bash
less Sup3rS3cretPickl3Ingred.txt   # more / head / tail / grep . all work equally well
```

**Result:** The first ingredient was recovered. *(Value intentionally omitted; solve the room.)*

---

## Phase 3: Privilege Escalation

### Assessing `sudo` Rights
The decisive check took seconds:

```bash
sudo -l
# User www-data may run the following commands:
#     (ALL) NOPASSWD: ALL
```

`(ALL) NOPASSWD: ALL` means the web service account can execute any command as root without a password: a complete failure of least privilege, and effectively game over for the host.

### Error & Correction 2: Reading an Awkward Filename
* **The Action:** The second ingredient lived in `/home/rick/` in a file whose name contained a space. Reading it directly rendered poorly through the browser-based panel.
* **The Correction:** Encoding the file to Base64 forces the bytes through the web display intact; it is then decoded locally.

```bash
sudo base64 "/home/rick/second ingredients"
# decode locally:
echo '<base64-string>' | base64 -d
```

### Reaching root's Home
The final ingredient followed the usual pattern of residing in root's home directory, now trivially readable with full `sudo`:

```bash
sudo ls -la /root
sudo less /root/3rd.txt
```

All three ingredients were recovered, and the box was fully compromised.

---

## Business & Operational Risk Impact
This chain maps cleanly to fixable control failures:
* **Excessive Privilege (`sudo NOPASSWD: ALL`):** The single most severe issue. Any code execution as `www-data` became instant root. Service accounts must be scoped to specific, non-shell-spawning binaries and require authentication.
* **Unrestricted Command Execution:** A web-facing OS-command interface should not exist; where unavoidable, a blocklist (filtering `cat`) is security theater and must be replaced by a fixed allowlist of parameterized actions.
* **Information Disclosure:** Hardcoded credentials in source comments and `robots.txt` gated the entire application behind a single, publicly readable pair.
* **Business Impact:** In a production or regulated environment, this path would allow an attacker to read protected records, tamper with data integrity, or establish persistence: a full confidentiality, integrity, and availability compromise from an unauthenticated starting point.
