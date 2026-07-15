# H0n3yP0t-S13M-H0M3LAB
[README.md](https://github.com/user-attachments/files/30042995/README.md)
# Honeypot & SIEM Home Lab

An internet-exposed SSH honeypot feeding a self-hosted SIEM, built to collect real-world attack telemetry and practise detection engineering. A VPS runs [Cowrie](https://github.com/cowrie/cowrie) to capture brute-force and post-exploitation activity; logs are shipped over a private WireGuard tunnel to a [Wazuh](https://wazuh.com/) server running on home-lab hardware, where they are decoded, alerted on, and mapped to MITRE ATT&CK.

> **Status: 🚧 in progress.** This is an active build. The architecture and detection design below reflect the target state; the **Build progress** section tracks what is actually deployed and verified. Attack statistics are added only once the honeypot is collecting live data.

---

## Why this project

I'm moving from carrier-grade network operations into cybersecurity, and this lab is where the theory from my studies meets real adversary behaviour. Rather than replay a canned dataset, the honeypot sits on a public IP and records what the internet actually throws at an exposed SSH port — then I build the detections that catch it. The write-up doubles as a build log so the reasoning behind each decision is visible, not just the finished config.

## Architecture

```
                    Internet (attackers)
                            │
                            ▼  port 22 (bait)
        ┌───────────────────────────────────┐
        │            VPS (Ubuntu 24.04)      │
        │                                    │
        │   Cowrie honeypot  ──► JSON logs   │
        │   Real SSH admin on port 2222      │
        │   Wazuh agent                      │
        └──────────────┬────────────────────┘
                       │  WireGuard tunnel
                       │  (VPS is public endpoint;
                       │   home dials out — Starlink CGNAT)
                       ▼
        ┌───────────────────────────────────┐
        │      Home lab (HP Pavilion)        │
        │                                    │
        │   Wazuh manager + indexer + dash   │
        │   Detection rules, ATT&CK mapping  │
        │   Alerts ──► Telegram              │
        └───────────────────────────────────┘
```

### Design decisions

- **Real SSH is moved to a non-standard port** so that port 22 can be handed entirely to the honeypot. Anything hitting 22 is, by definition, not me.
- **The Wazuh server lives at home, not on the VPS.** Keeping the SIEM off the exposed host means an attacker who compromises the honeypot never has line of sight to the log store or the detections written against them. The VPS is treated as untrusted.
- **WireGuard is dialled out from home to the VPS.** The home connection is behind Starlink CGNAT with no inbound reachability, so the tunnel is established from the home side; the VPS acts as the fixed public endpoint the agent traffic rides back through.
- **The indexer heap is capped** so the Wazuh stack coexists with an existing local LLM inference workload on the same home machine.

## Components

| Component | Role | Host |
|-----------|------|------|
| Cowrie | Medium-interaction SSH honeypot; logs credentials tried and commands run | VPS |
| Wazuh agent | Ships Cowrie JSON and host telemetry to the manager | VPS |
| WireGuard | Encrypted transport between VPS and home lab | Both |
| Wazuh manager | Rule engine, decoders, alerting | Home lab |
| Wazuh indexer + dashboard | Storage and visualisation | Home lab |

## Detection engineering

The point of the lab is the detections, not the honeypot itself. Planned coverage, mapped to MITRE ATT&CK:

- **Credential brute force (T1110)** — alert on repeated failed authentications from a single source within a short window.
- **Valid-account use (T1078)** — flag a "successful" login into the Cowrie sandbox, i.e. an attacker who got past the (deliberately weak) credentials.
- **Post-login command capture** — decode the commands attackers run inside the sandbox and surface the common patterns (payload downloads, persistence attempts, recon).
- **Custom Cowrie decoder** — parse Cowrie's JSON fields (source IP, username, password, input) into Wazuh so rules can match on them directly.
- **Purple-team validation** — run scripted ATT&CK techniques against an agent host with [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team) and confirm each fires the expected alert, pairing *technique executed → alert raised*.

## Build progress

- [x] Provision VPS (Ubuntu 24.04 LTS)
- [ ] Host hardening — SSH to non-standard port, key-only auth, UFW, fail2ban
- [ ] WireGuard tunnel: VPS ↔ home lab
- [ ] Wazuh server (manager + indexer + dashboard) on home lab
- [ ] Enrol Wazuh agents (VPS + home hosts)
- [ ] Deploy Cowrie on the VPS
- [ ] Ship Cowrie logs into Wazuh (custom decoder + rules)
- [ ] MITRE ATT&CK rule mapping
- [ ] Atomic Red Team purple-team validation
- [ ] Telegram alerting on high-severity events
- [ ] Publish 30-day honeypot analysis

## Repository layout

*(populated as the build progresses)*

```
wazuh/
  rules/          custom detection rules
  decoders/       Cowrie JSON decoder
cowrie/           honeypot config (sanitised)
wireguard/        tunnel config templates (no keys)
docs/             architecture notes, build log, analysis
```

## Notes on safety and scope

This is a personal research lab. The honeypot is a sandbox by design and is not connected to any production system. Configuration published here is sanitised — no keys, tokens, or real IPs. Attacker source data shown in any analysis is aggregated and used only to characterise attack patterns.

---

*Part of an ongoing home-lab portfolio. Companion repos: [t2-macbook-linux](https://github.com/LVZFR), [home-network-monitoring](https://github.com/LVZFR).*
