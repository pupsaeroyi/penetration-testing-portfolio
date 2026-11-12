# Kali Linux: reusable shell and enumeration notes

Learned in [HTB Cap](../htb-writeups/machines/cap/README.md). These Bash/Linux utilities are not exclusive to Kali. Local setup runs on your workstation; target enumeration runs inside the authorized target shell.

## Local workspace (Kali / Bash)

```bash
mkdir -p ~/htb/cap/scans
cd ~/htb/cap
export IP=10.129.168.241
echo "$IP"
```

Replace the machine name and temporary IP for each lab. Do not put angle brackets around an IP: `<` and `>` are shell redirection operators. This Linux workspace is separate from the Windows portfolio repository.

## Connect to services

```bash
ftp "$IP"
ssh "nathan@$IP"
```

Replace `nathan` with the authorized lab account. Enter passwords at the interactive prompt rather than including them in commands or public notes.

Inside the FTP client:

```text
pwd
ls -la
get filename.txt
bye
```

`get` downloads the specified remote file to your local directory. Replace `filename.txt` with the intended lab file.

## Identify the target context

After obtaining a target shell:

```bash
whoami
id
hostname
pwd
cat /etc/os-release
uname -a
```

These identify the account, UID/GID and groups, host, directory, OS, and kernel. The supplied Cap walkthrough lists `uname -a` but does not document its output.

## Enumerate privileges

```bash
sudo -l
find /usr/bin /usr/local/bin -type f -perm -4000 2>/dev/null
getcap -r / 2>/dev/null
```

| Command | Purpose and limits |
|---|---|
| `sudo -l` | List permitted sudo commands; may request a password. |
| `find ... -perm -4000` | Find SUID files in the two listed directories, not the whole filesystem. |
| `getcap -r /` | Search recursively for file capabilities; may take time and encounter inaccessible paths. |

`2>/dev/null` suppresses errors, so missing output does not prove every path was inspected. SUID files and capabilities require interpretation; their presence alone does not prove exploitability.

In Cap, Python's `CAP_SETUID` was the decisive misconfiguration. The exact escalation command, prerequisites, and proof belong in the lab reports.

## Execute Python through its interpreter

```bash
python3 -c 'import os; print(os.getuid())'
```

`-c` executes a Python source string. Python expressions such as `os.setuid(0)` are not standalone Bash commands. Interpreter paths and permissions vary by host.

After a privilege change, use `id` to verify the identity; a shell prompt alone is insufficient proof.

## NFS enumeration

In an authorized lab, list exports and mount an available export read-only for inspection:

```bash
showmount -e "$IP"
sudo mkdir -p /mnt/lab-nfs
sudo mount -t nfs -o ro,vers=3,proto=tcp,mountproto=tcp "$IP:/export/path" /mnt/lab-nfs
ls -la /mnt/lab-nfs
sudo umount /mnt/lab-nfs
```

Replace `/export/path` with the discovered export. NFS versions and mount options depend on the server. Documents on a share can contain sensitive information, so keep credentials out of published notes.

## SSH local port forwarding

```bash
ssh -N -L 1337:127.0.0.1:1337 user@target
```

This connects local port 1337 to port 1337 on the SSH server's loopback interface. Replace the account, target, and ports as needed. `-N` keeps the connection open without starting a remote shell. Browse to `http://127.0.0.1:1337` locally when the forwarded service is HTTP.

## Password hashes and interactive shells

In an authorized password audit, John the Ripper can test a wordlist against hashes in a local file. Protect both the hash file and recovered passwords.

```bash
john --wordlist=/path/to/wordlist hashes.txt
john --show hashes.txt
```

Supported hash formats and input requirements depend on the installed version. A bcrypt prefix such as `$2y$` indicates a password hash, and a recovered password should be validated only within the authorized scope.

A limited shell may need a pseudoterminal for interactive programs such as `su`:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

The [Enigma walkthrough](../htb-writeups/machines/enigma/HTB_Enigma_Full_Learning_Walkthrough.md) records the lab-specific use of these techniques.
