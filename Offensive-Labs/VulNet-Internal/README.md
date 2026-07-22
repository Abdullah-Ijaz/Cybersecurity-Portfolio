# VulNet: Internal (TryHackMe)

**Platform:** TryHackMe  **Difficulty:** Medium  **Category:** Internal Network / Lateral Movement
**Scope:** Internal black-box, network access only, no starting credentials
**Status:** *In progress; documented through the TeamCity foothold, root pending.*

## Executive Summary & Methodology
This box is a chain of internal services where no single step is the whole exploit; each failed attempt points at the next one. The path that worked came from abandoning a vector that stalled rather than forcing it: an NFS mount was denied, so I pivoted to SMB, which handed over config files, which led to Redis, which held the credentials for rsync, which put files on the target. I document the missteps and the recovery from each, because on an internal engagement the pivot matters as much as the payload.

---

## Phase 1: Initial Enumeration & Troubleshooting

### The Initial Scan
I started with a full-port Nmap scan to define the attack surface.

```bash
nmap -sV -sC -p- -T4 <Target_IP>
```
**Critical Findings:**
* **Port 111 (rpcbind) & 2049 (nfs):** Indicators of network file sharing.
* **Port 139/445 (Samba):** Windows-style file sharing on a Linux host.
* **Port 873 (rsync):** File synchronization service.
* **Port 6379 (Redis):** In-memory data store.

### Error & Correction 1: The NFS Roadblock
My first move was to mount the NFS share directly.
* **The Action:** `sudo mount -t nfs <IP>:/opt/internal /tmp/mount1 -nolock`
* **The Error:** `mount.nfs: access denied by server`
* **The Pivot:** Rather than force the NFS vector, I shifted to SMB (Port 445) to find another way in. `smbclient -L //<IP> -N` enumerated the shares; the `shares` directory gave up `services.txt` and `business-req.txt`, which provided the initial situational awareness I needed.

---

## Phase 2: Re-Assessment & Exploitation

### Error & Correction 2: Correcting the NFS Target
The `services.txt` detail showed my first NFS attempt had targeted the wrong export path. I corrected the command to the specific configuration directory:

```bash
sudo mount -t nfs <IP>:/opt/conf /mnt/nfs_exploit -o nolock
```
**Success:** The mount succeeded.

*Risk Analysis Note: Exposing configuration directories over an internal NFS export is a serious compliance problem (HIPAA/PCI-DSS territory). Config files routinely carry hardcoded credentials and database paths, which is exactly the lateral-movement fuel an internal attacker wants.*

### Extracting Database Credentials
In the mounted config directory I went straight for the Redis configuration and pulled the auth key with grep:

```bash
grep -i "pass" redis.conf
# Result: requirepass "<redacted>"
```

---

## Phase 3: Data Extraction & Syntax Troubleshooting

With the password in hand, I authenticated to the internal Redis service on Port 6379.

### Error & Correction 3: The "Wrong Type" Error
Several keys were present, including `internal flag` and `authlist`.
* **The Action:** I tried to read the list with a standard string command: `get authlist`
* **The Error:** `(error) WRONGTYPE Operation against a key holding the wrong kind of value`
* **The Correction:** `authlist` is a Redis list, not a string, so I switched to the list command:

```bash
LRANGE authlist 0 -1
```
**Result:** This dumped four Base64 strings. Decoding them gave the authorization credentials for the rsync service: `rsync-connect@127.0.0.1` with password `<redacted>`.

---

## Phase 4: Current Roadblock & Next Steps (TeamCity Shell)

With the rsync credentials, I pulled the internal system files down to a local `~/rsync_loot` directory.

**The attack chain so far:**
1. Generated a custom SSH key and uploaded it to the target over rsync.
2. Opened an SSH tunnel from my machine to the target server.
3. Found a superuser authentication token in the TeamCity logs.
4. Authenticated to the TeamCity web portal through the browser over the tunnel.

**Current status:**
I tried to open a reverse shell by injecting a Python payload into TeamCity, but my local Netcat listener never caught the connection.

**Next diagnostic steps:**
* Check the Python payload syntax against the target's specific Python version.
* Confirm whether the target firewall is blocking outbound connections on my listener port.
* Try a different delivery method inside TeamCity (a bash script rather than a raw Python one-liner).

---

## Business & Operational Risk Impact
This chain represents a serious failure of internal segmentation and secrets handling.
* **Improper access control (NFS):** An over-permissive export allowed unauthenticated reading of critical configuration files.
* **Hardcoded credentials:** The Redis password sat in plaintext in a config file, which opened a direct path to the internal data store and everything downstream of it.
* **Business impact:** In a live environment, this internal pivot would let an attacker reach protected records, tamper with data, or stage network-wide ransomware. Each hop here was a service that trusted the previous one by default, which is how a single misconfigured export turns into full internal compromise.
