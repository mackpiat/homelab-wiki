# CrowdSec IDS/IPS Deployment

## Overview
This document outlines the deployment and configuration of CrowdSec on the Debian router. CrowdSec acts as a collaborative Intrusion Detection System (IDS) and Intrusion Prevention System (IPS). It parses local system logs in real-time, cross-references suspicious behavior against a global threat intelligence network, and actively drops malicious packets at the network edge.

## Architecture
* **Core Engine** - `crowdsec` (Responsible for log acquisition and parsing)
* **Mitigation Bouncer** - `crowdsec-firewall-bouncer-nftables` (Responsible for manipulating firewall rules)
* **Telemetry** - `cscli` (Command Line Interface for engine management)

## 1. Installation

The deployment requires two distinct components - the monitoring agent and the enforcement bouncer.

- Add the official CrowdSec repository to receive timely security patches directly from the maintainers, bypassing older Debian repos.
```bash
curl -s [https://install.crowdsec.net](https://install.crowdsec.net) | sudo bash
```

- Install the core engine. By default, it runs as a daemon but does not actively block traffic yet.
```bash
sudo apt install crowdsec -y
```

- Install the specific bouncer for our firewall backend. This binary reads the local API decisions from the CrowdSec engine and automatically creates 'drop' rules in the nftables chains.
```bash
sudo apt install crowdsec-firewall-bouncer-nftables -y
```
## 2. Threat Intelligence Sync
- To ensure the router defends against zero-day threats and currently active botnets, the local hub must be synchronized with the global database.
- 'hub update' pulls the latest index of available parsers and global blocklists.
- 'hub upgrade' applies them to the local engine, immediately shielding the router from tens of thousands of globally recognized malicious IPs.
```bash
sudo cscli hub update && sudo cscli hub upgrade
```

## 3. Log Acquisition & Parsing (The systemd Fix)
By default, modern Debian systems utilize journald for logging, rather than flat files in /var/log/. The engine required specific parsers to translate the systemd journal format into actionable threat intelligence.

- Install the baseline Linux collection, providing the dictionaries necessary to understand syslog and iptables/nftables drop logs.
```bash
sudo cscli collections install crowdsecurity/linux
```

- Install the specific dictionaries required to parse OpenSSH daemon logs, allowing the engine to detect SSH brute-force and exploit attempts.
```bash
sudo cscli collections install crowdsecurity/sshd
```
Baseline Linux and sshd collection comes along with the package (Optional to run the above commands, incase the collections are not installed).

- The CrowdSec daemon loads parsers into memory at boot. Restarting the service forces it to apply the newly installed collections to the live journal stream.
```bash
sudo systemctl restart crowdsec
```

## 4. Operational Verification
- View all IPs currently banned by the local firewall
```bash
sudo cscli decisions list
```

- View the status of log parsing and acquisition (Verify 'Lines parsed' increments under Acquisition Metrics)
```bash
sudo cscli metrics
```

- Tail the live CrowdSec engine logs via the systemd journal
```bash
sudo journalctl -u crowdsec -f
```
