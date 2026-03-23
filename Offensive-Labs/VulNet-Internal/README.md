## Executive Summary & Methodology
In penetration testing, much like in incident response or financial risk management, initial assessments often require rapid course correction. Success relies on remaining analytical, recognizing when a diagnostic approach fails, and pivoting to a new methodology. 

This report documents the internal enumeration and exploitation of a compromised server, explicitly highlighting the missteps, troubleshooting steps, and critical thinking required to navigate an internal network.

---

## Phase 1: Initial Enumeration & Troubleshooting

### The Initial Scan
The assessment began with a targeted Nmap scan to identify the attack surface.

```bash
nmap -sV -sC -p- -T4 <Target_IP>
```
**Critical Findings:**
* **Port 111 (rpcbind) & 2049 (nfs):** Indicators of network file sharing.
* **Port 139/445 (Samba):** Windows-style file sharing on a Linux host.
* **Port 873 (rsync):** File synchronization service.
* **Port 6379 (Redis):** In-memory data store.

### Error & Correction 1: The NFS Roadblock
The initial approach involved mounting the NFS share directly to the root directory.
* **The Action:** `sudo mount -t nfs <IP>:/opt/internal /tmp/mount1 -nolock`
* **The Error:** `mount.nfs: access denied by server`
* **The Pivot:** Paused the NFS vector and shifted focus to the SMB service (Port 445) to identify an alternative entry point. Using `smbclient -L //<IP> -N`, the shares were enumerated. Accessing the `shares` directory allowed the extraction of `services.txt` and `business-req.txt`, providing initial situational awareness.

---

## Phase 2: Re-Assessment & Exploitation

### Error & Correction 2: Correcting the NFS Target
Reviewing the initial NFS failure indicated the wrong export path was targeted. The command was corrected to target the specific configuration directory.

```bash
sudo mount -t nfs <IP>:/opt/conf /mnt/nfs_exploit -o nolock
```
**Success:** The mount succeeded. 

*Risk Analysis Note: Exposing configuration directories internally is a critical compliance violation (e.g., HIPAA/PCI-DSS). These files frequently contain hardcoded credentials and database paths that facilitate lateral movement.*

### Extracting Database Credentials
Navigating the mounted directory, Redis configuration files were targeted. Using grep, the authentication key was isolated:

```bash
grep -i "pass" redis.conf
# Result: requirepass "B65Hx562F@ggAZ@F"
```

---

## Phase 3: Data Extraction & Syntax Troubleshooting

With the credentials secured, authentication into the internal Redis service on Port 6379 was successful. 

### Error & Correction 3: The "Wrong Type" Error
Several keys were identified, including `internal flag` and `authlist`. 
* **The Action:** Attempted to read the list using a standard string command: `get authlist`
* **The Error:** `(error) WRONGTYPE Operation against a key holding the wrong kind of value`
* **The Correction:** Recognizing that `authlist` is a list data structure, the syntax was adjusted:

```bash
LRANGE authlist 0 -1
```
**Result:** This command successfully dumped four Base64 encoded strings. Decoding these revealed authorization credentials for the Rsync service: `rsync-connect@127.0.0.1` with password `Hcg3HP67@TW@Bc72v`.

---

## Phase 4: Current Roadblock & Next Steps (TeamCity Shell)

Armed with the Rsync credentials, the internal system files were downloaded to a local `~/rsync_loot` directory. 

**The Attack Chain So Far:**
1. Generated a custom SSH key and uploaded it to the target via Rsync.
2. Established an SSH tunnel from the local machine to the target server.
3. Located a superuser authentication token (`2206011471080847658`) in the TeamCity logs.
4. Successfully authenticated into the TeamCity web portal via the browser using the tunneled connection.

**Current Status:**
An attempt was made to establish a reverse shell by injecting a Python payload into TeamCity. However, the local Netcat listener failed to catch the connection. 

**Next Diagnostic Steps:**
* Verify the Python payload syntax against the target's specific Python version.
* Check if the target firewall is blocking outbound connections on the chosen listening port.
* Explore alternative payload delivery methods within TeamCity (e.g., executing a bash script instead of a raw Python one-liner).

---

## Financial & Clinical Risk Impact
The vulnerabilities exploited in this chain represent a severe failure of internal security protocols. 
* **Improper Access Control (NFS):** Allowed unauthenticated reading of critical configuration files.
* **Hardcoded Credentials:** Storing the Redis password in plaintext provided direct access to internal databases.
* **Business Impact:** In a live environment, an attacker could leverage this internal pivot to access protected records, manipulate financial data, or deploy network-wide ransomware.
