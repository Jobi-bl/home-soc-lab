# Home SOC Lab

I built this lab to actually learn how SOC detection works instead of just reading about it. I'm trying to break into cybersecurity as an entry-level SOC analyst, and I figured the best way to understand alerts, logs, and detection rules is to set up my own environment, attack it myself, and see what shows up on the other side.

Everything here is real — no cloud shortcuts, no pre-built templates. I ran into a bunch of actual problems while setting this up (firewall misconfig, enrollment failures, SSH rate-limiting fighting back against my own attacks) and I've documented those too, because troubleshooting is honestly half of what a SOC analyst does anyway.

## What's in this lab

- **Wazuh SIEM** (manager + dashboard) — this is where all the logs land and get correlated into alerts
- **Kali Linux** — used as the attacker box to generate real traffic to detect
- **Ubuntu Server** — the monitored endpoint, running the Wazuh agent

All three VMs sit on the same bridged network so they behave like real hosts on a LAN, not isolated NAT boxes.

## Why I set it up this way

I originally tried Internal Network mode in VirtualBox to keep things isolated, but switched to bridged so the machines would behave more like a real network segment — and honestly also because I'd already hit some bridged adapter issues earlier and wanted to actually fix that properly instead of avoiding it.

## What I've done so far

### 1. Getting the agent connected
This wasn't as simple as installing and walking away. The agent kept failing to enroll with the manager (`Unable to connect to enrollment service`), and it took some digging through `ossec.log` and checking listening ports with `ss -tulnp` to confirm the manager was actually reachable before it finally connected. Turns out ping and basic connectivity were fine the whole time — it just needed a retry cycle to complete enrollment properly.

### 2. Nmap scan against the target
Ran a basic service detection scan from Kali:
```
nmap -sV 10.132.159.111
```
Found SSH open, running OpenSSH 10.2p1 on Ubuntu. Nothing wild here, but it's the first real check that the target is reachable and confirms what's exposed.

### 3. SSH brute-force attempt with Hydra
```
hydra -l root -P /usr/share/wordlists/rockyou.txt -t 4 ssh://10.132.159.111
```
This one didn't go the way I expected. OpenSSH's own rate-limiting (`MaxAuthTries` / preauth throttling) kicked in almost immediately and started dropping Hydra's connections outright — Hydra actually threw `[ERROR] all children were disabled due too many connection errors` before it could get far into the wordlist.

Even though the attack got throttled, Wazuh still caught everything:
- **468 authentication failure events** logged and correlated
- Automatically mapped to MITRE ATT&CK techniques: **Password Guessing, Brute Force, SSH**
- Alert levels ranged 5–10 (medium severity)

I also noticed 21 "authentication success" events in the dashboard and wanted to make sure I wasn't misreading a compromise that didn't happen. Checked the actual events — every single one was rule 5501 ("PAM: Login session opened"), which is just normal login session activity from my own admin sessions on the Wazuh and target boxes, completely unrelated to the Hydra attempt. Good reminder to actually verify what an alert is showing instead of trusting a total count at face value.

## What I learned

- Rate-limiting on a target can mess with attack tools in ways that actually make for a more interesting detection story than a clean successful brute force would have
- Wazuh's default ruleset picks up SSH auth failures without any custom rule writing — MITRE mapping happens automatically
- Alert totals need to be checked against actual event details before drawing conclusions — the 21 "successes" almost got misreported as something they weren't

## What's next

- Adding a Windows endpoint with Sysmon for richer telemetry (process creation, registry changes)
- More exercise types — file integrity monitoring, malware detection test (EICAR), maybe a privilege escalation attempt
- Suricata for network-level detection alongside the log-based stuff Wazuh already covers
- A proper network diagram

## Notes

Screenshots and detailed exercise write-ups are in their own folders. I'm updating this as I go rather than dumping it all at once, since the actual point of this project is to show the process, not just the end result.
