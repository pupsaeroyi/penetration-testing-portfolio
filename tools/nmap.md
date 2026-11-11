# Nmap — reusable notes

Learned in [HTB Cap](../htb-writeups/machines/cap/README.md). Keep each machine's actual output in its own `scans/` folder.

## Initial service enumeration

Run in your Kali/Bash lab workspace after setting `IP` to the current authorized target:

```bash
mkdir -p scans
nmap -sC -sV -oN scans/initial_scan.txt "$IP"
```

| Option | Purpose |
|---|---|
| `-sC` | Run default NSE scripts; ensure these are within assessment scope. |
| `-sV` | Probe services to identify software and versions. |
| `-oN` | Save human-readable output. |
| `"$IP"` | Use the address stored in your shell variable. |

This scans the default selection of 1,000 TCP ports, not every TCP port or UDP. An open port is a service clue, not proof of a vulnerability.

## Use results to choose next steps

- FTP: investigate permitted authentication and file access.
- SSH: consider shell access if valid lab credentials are obtained.
- HTTP: inspect routes, downloads, authentication, and authorization.

Add new reusable syntax here once; keep addresses, full outputs, and machine-specific reasoning in each lab's reports.
