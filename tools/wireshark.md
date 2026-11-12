# Wireshark: reusable notes

Learned in [HTB Cap](../htb-writeups/machines/cap/README.md). These are display filters and UI actions, not shell commands.

## Open a saved capture

Use **File → Open** (`Ctrl+O`) and select the PCAP. You do not need to start a live capture to analyze a saved file.

## Display filters

Enter one filter at a time in the display-filter bar:

| Filter | Shows |
|---|---|
| `ftp` | Decoded FTP control traffic. |
| `http` | Decoded HTTP traffic. |
| `tcp` | TCP traffic. |
| `tcp.port == 21` | TCP packets with source or destination port 21. |
| `ip.addr == 192.168.196.16` | Traffic to or from this example address; replace it for your capture. |
| `tcp.stream eq 3` | One TCP conversation; replace `3` with the relevant stream index. |

Filters change the visible packets, not the saved capture. Stream indices are specific to each capture.

## Follow a conversation

1. Filter for a protocol such as `ftp`.
2. Select a packet in the conversation.
3. Right-click → **Follow → TCP Stream**.
4. Read the requests and responses together.

In plaintext FTP, `USER` identifies an account and `PASS` can expose its password. Response `230` means authentication succeeded at capture time; `530` means it was rejected. A subsequent file-operation failure does not necessarily mean login failed.

Keep specific discoveries and screenshots in the lab folder. Redact passwords, tokens, and flags from published evidence; PCAPs can contain sensitive payloads.
