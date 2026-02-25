# T470 Edge Router Infrastructure

Welcome to the internal documentation wiki for the homelab edge environment.

## Overview
This system serves as the primary gateway, DNS resolver, and active defense node for the network. It is built on a "trust no one" architecture.

## Core Services
* **Hardware:** Lenovo ThinkPad T470 (Utilizing battery as built-in UPS)
* **OS:** Debian Linux
* **Perimeter Defense:** CrowdSec IDS/IPS (nftables bouncer)
* **DNS Resolution:** Unbound (Recursive caching) + AdGuard Home
* **Overlay Network:** Tailscale

---
*Documentation maintained via automated CI/CD pipeline.*
