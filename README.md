# Penetration Testing Portfolio

I am a fourth-year Information and Communication Engineering (ICE) student at Chulalongkorn University building practical skills for a career in penetration testing.

This repository documents my work in authorized training labs: how I enumerate services, investigate weaknesses, validate impact, and explain remediation. Each lab has a professional-style assessment report and a separate learning walkthrough with commands, decisions, mistakes, and lessons.

## Featured lab

| Lab | Environment | Focus | Reports |
|---|---|---|---|
| [HTB Cap](htb-writeups/machines/cap/README.md) | Easy / Linux | Service enumeration, packet analysis, exposed FTP credentials, Linux file capabilities | [Assessment](htb-writeups/machines/cap/HTB_Cap_Professional_Pentest_Report.md) · [Learning walkthrough](htb-writeups/machines/cap/HTB_Cap_Full_Learning_Walkthrough.md) |

Cap's reports document standard-user access and a verified root shell. Root flag retrieval/submission is not documented. Supporting raw evidence files are not yet included.

## Reusable tool notes

- [Nmap](tools/nmap.md): service discovery and interpreting scan results.
- [Wireshark](tools/wireshark.md): display filters and reconstructing conversations.
- [Kali Linux and shell commands](tools/kali-linux.md): workspace setup, remote access, and Linux privilege enumeration.

Tool notes capture reusable syntax. Each machine's reports retain its specific findings and reasoning.

## Repository structure

```text
htb-writeups/
  machines/           Machine reports and supporting artifacts
  starting-point/     Reserved for introductory labs
tools/                Reusable tool notes
web-security/         Reserved for web vulnerability notes
security-projects/    Reserved for independent assessment projects
scripts/              Reserved for scripts developed during practice
```

Reserved folders are organizational placeholders, not completed projects.

## Reporting approach

- **Professional report:** scope, executive summary, findings, supporting evidence, impact, remediation, and limitations.
- **Learning walkthrough:** commands, hypotheses, unsuccessful attempts, explanations, and transferable lessons.

All assessments here concern authorized training environments. They are learning exercises, not client engagements. Reported outcomes distinguish verified results from untested assumptions. Published material excludes passwords and flags; lab writeups contain spoilers.
