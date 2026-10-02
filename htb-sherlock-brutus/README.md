# Personal Write-up: Sherlock Brutus (HTB)

This first Sherlock introduces Unix auth logs and the WTMP binary log format.
It marks the beginning of my blue team learning path.

## Scenario

On 6 March 2024, a Confluence server was brute-forced over SSH from the IP
address 65.2.161.68. The root account was compromised, and the attacker then
created a new user and added it to the sudo group to maintain persistence.
The attacker also downloaded Linper, a Linux persistence tool, using curl.

## Attack Timeline

| Time (UTC) | Event | Phase | Evidence |
|---|---|---|---|
| 06:19:54 | Legitimate root login from 203.101.190.9 | Baseline (not malicious) | auth.log |
| 06:31:33 – 06:31:42 | 48 failed SSH logins from 65.2.161.68 against 5 accounts (server_adm 12, svc_account 11, admin 10, backup 9, root 6) | Brute force | auth.log |
| 06:31:40 | Successful SSH login as root from 65.2.161.68 — brute force succeeded | Initial access | auth.log + wtmp |
| 06:32:44 – 06:37:24 | Interactive root session from 65.2.161.68 | Execution | auth.log + wtmp |
| 06:34:18 | Group and user cyberjunkie created (UID/GID 1002) | Persistence | auth.log |
| 06:35:15 | cyberjunkie added to the sudo group | Privilege escalation | auth.log |
| 06:37:34 | Successful SSH login as cyberjunkie from 65.2.161.68 | Valid accounts | auth.log + wtmp |
| ~06:37:57 | sudo cat /etc/shadow | Credential access | auth.log (sudo COMMAND) |
| 06:39:38 | sudo curl linper.sh from raw.githubusercontent.com | Persistence | auth.log (sudo COMMAND) |

## Indicators of Compromise

| Type | Value | Notes |
|---|---|---|
| IPv4 | 65.2.161.68 | Source of the brute force and of every malicious session |
| URL | https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh | Persistence tool downloaded by the attacker. Block the full URL, not the domain: raw.githubusercontent.com hosts legitimate content |
| Account | cyberjunkie (UID 1002, GID 1002) | Backdoor account created by the attacker and added to the sudo group |
| Usernames | root, admin, backup, server_adm, svc_account | Accounts targeted during the brute force — useful to spot the same campaign on other hosts |

## MITRE ATT&CK Mapping

| Time (UTC) | Tactic | Technique | ID |
|---|---|---|---|
| 06:31:33 – 06:31:42 | Credential Access | Brute Force: Password Guessing | T1110.001 |
| 06:31:33 – 06:39:38 | Lateral Movement | Remote Services: SSH | T1021.004 |
| 06:31:40 | Initial Access | Valid Accounts: Local Accounts | T1078.003 |
| 06:34:18 | Persistence | Create Account: Local Account | T1136.001 |
| 06:35:15 | Privilege Escalation | Account Manipulation: Additional Local or Domain Groups | T1098.007 |
| 06:37:34 | Persistence | Valid Accounts: Local Accounts | T1078.003 |
| 06:37:57 | Credential Access | OS Credential Dumping: /etc/passwd and /etc/shadow | T1003.008 |
| 06:39:38 | Command and Control | Ingress Tool Transfer | T1105 |

## Methodology & Tools

### Environment
I analysed the evidence with Splunk Enterprise 10.4.3, installed on an
Ubuntu 22.04 virtual machine hosted on my personal Proxmox lab.

### Data ingestion
HTB provided three files: auth.log, utmp.py, wtmp

auth.log was uploaded with the linux_secure sourcetype and indexed into a dedicated
index named sherlock_brutus.

wtmp is a binary file and cannot be read directly. I converted it using the utmp.py script provided with the challenge.

### Assumptions

Syslog lines do not carry the year. Splunk assigned the current
year (2026) at ingestion time, while the incident actually took place in
March 2024.

### Key searches
