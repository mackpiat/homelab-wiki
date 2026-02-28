# Unbound Recursive DNS Resolver

## Overview
This document outlines the configuration of **Unbound**, deployed as a fully recursive, authoritative DNS resolver on the Debian router.

By default, most networks forward DNS queries to upstream corporate providers (like Cloudflare's `1.1.1.1` or Google's `8.8.8.8`). This setup eliminates that dependency. When a client requests a domain, Unbound queries the 13 internet Root Servers directly, traverses to the Top Level Domain (TLD) servers, and finally to the authoritative nameservers.

**Result:** Maximum privacy. No single third-party entity possesses a complete log of the network's DNS queries.

## Architecture & Traffic Flow
1. Client device requests a domain.
2. The query is routed through the Tailscale tunnel to the **AdGuard Home** instance.
3. AdGuard filters the request against known ad/tracker blocklists.
4. If permitted, AdGuard forwards the request locally to **Unbound** (`127.0.0.1:5335`).
5. Unbound performs the recursive resolution and caches the result for future identical queries.

## Installation & Root Hints

First, we install the service and download the `root.hints` file. This file acts as the ultimate map, telling Unbound the IP addresses of the 13 master DNS servers that govern the internet.

- Unbound is available directly in the main Debian repositories.
```bash
sudo apt install unbound -y
```

- We must manually download the latest list of root servers from InterNIC. The '-O' flag directs wget to overwrite the existing, potentially outdated `root.hints` file that comes with the default installation.
```bash
wget [https://www.internic.net/domain/named.cache](https://www.internic.net/domain/named.cache) -qO /var/lib/unbound/root.hints
```

## Core Configuration
The primary configuration is written to a dedicated file to avoid modifying the base Unbound template. This ensures updates to the package do not overwrite our custom routing logic.

- Create and edit the custom configuration file
```bash
sudo vim /etc/unbound/unbound.conf.d/homelab.conf
```
### Configuration Paylod
```
server:
    # Increase the verbosity (L1) for troubleshooting.
    verbosity: 1

    # Point Unbound to the map of the internet thats downloaded
    root-hints: "/var/lib/unbount/root.hints"

    # AdGuard Home must operate on standard port 53 to intercept client network requests.
    # Therefore, Unbound is shifted to port 5335 to prevent a bind-address conflict.
    port: 5335

    # Unbound should only accept queries forwarded by the local AdGuard instance.
    # Listening strictly on the loopback interface prevents the service from being
    # weaponized in a DNS amplification attack from the open internet.
    interface: 127.0.0.1

    # Limits the service to IPv4 only. Disabling IPv6 prevents unnecessary
    # external queries and potential DNS leaks, as the internal homelab does not utilize IPv6.
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: no

    # Enables DNSSEC validation. This ensures the DNS records received from
    # the root servers have not been cryptographically tampered with or hijacked in transit.
    harden-dnssec-stripped: yes

    # QNAME Minimisation is strictly for privacy. If a client asks for 'app.example.com',
    # Unbound only asks the root server where '.com' is, not the full string.
    # It minimizes metadata leaked to higher-level hierarchy servers.
    qname-minimisation: yes

    # Prevents DNS rebinding attacks. It ensures that malicious upstream servers
    # cannot return private, local IP addresses in their responses to hijack local routing.
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10

    # Caching optimization to serve expired cache entries immediately to the user while
    # unbound fetches the updated record in the background making the browsing feel instantly responsive.
    serve-expired: yes

    # Pre-fetching popular records before they expire from the cache.
    prefetch: yes
```

## Service Initialization
With the configuration written, the service is enabled to start on boot and restarted to apply the new parameters.

- 'enable --now' is a systemd shortcut that both enables the service to run at boot and starts it immediately in the current session.
```bash
sudo systemctl enable --now unbound
```

- Verify the service is running smoothly and actively listening on port 5335.
```bash
sudo systemctl status unbound
```

## AdGuard Integration
The final step bridges the two services. In the AdGuard Home web dashboard `(Settings -> DNS Settings)`, the Upstream DNS servers field is cleared of all external providers (e.g., Cloudflare, Google) and replaced exclusively with, `127.0.0.1:5335`.
