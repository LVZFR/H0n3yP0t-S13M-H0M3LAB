# Honeypot & SIEM Home Lab

An internet-exposed SSH honeypot feeding a self-hosted SIEM, built to collect real-world attack telemetry and practise detection engineering. A VPS runs [Cowrie](https://github.com/cowrie/cowrie) to capture brute-force and post-exploitation activity; logs are shipped over a private WireGuard tunnel to a [Wazuh](https://wazuh.com/) server running on home-lab hardware, where they are decoded, alerted on, and mapped to MITRE ATT&CK.

> **Status: 🚧 in progress.** The honeypot is live on a public IP and collecting data; detection rules are being written and validated. The **Build progress** section tracks what is actually deployed and verified. Attack statistics are added only once the 30-day collection window closes.

---

## Why this project

I'm moving from carrier-grade network operations into cybersecurity, and this lab is where the theory from my studies meets real adversary behaviour. Rather than replay a canned dataset, the honeypot sits on a public IP and records what the internet actually throws at an exposed SSH port — then I build the detections that catch it. The write-up doubles as a build log so the reasoning behind each decision is visible, not just the finished config.

## Architecture

```
                    Internet (attackers)
                            │
                            ▼  port 22 (bait, IPv4 + IPv6)
        ┌───────────────────────────────────┐
        │            VPS (Ubuntu 24.04)      │
        │                                    │
        │   Cowrie honeypot  ──► JSON logs   │
        │   Real SSH admin: tunnel-only      │
        │   Wazuh agent                      │
        └──────────────┬────────────────────┘
                       │  WireGuard tunnel
                       │  (VPS is public endpoint;
                       │   home dials out — Starlink CGNAT)
                       ▼
        ┌───────────────────────────────────┐         ┌──────────────────────────────┐
        │      Home lab (HP Pavilion)        │◄────────│   Test VM (Oracle Linux 9)   │
        │                                    │  agent  │                              │
        │   Wazuh manager + indexer + dash   │  over   │   Wazuh agent                │
        │   Detection rules, ATT&CK mapping  │  LAN    │   Atomic Red Team runner     │
        │   Alerts ──► Telegram              │         │   (purple-team validation)   │
        └───────────────────────────────────┘         └──────────────────────────────┘
              both on the home LAN (192.168.50.0/24); the VM is a disposable host —
              attacks run there, NOT on the honeypot or the manager.
```

### Design decisions

- **Real SSH is removed from the public internet entirely.** Rather than just moving it to a high port, `sshd` binds only to the WireGuard tunnel address (`10.66.66.1`); port 22 — on both IPv4 and IPv6 — is handed entirely to the honeypot via an interface-bound `iptables`/`ip6tables` redirect. Admin access is via an SSH jump host over the tunnel. Anything reaching a real shell prompt from the public side is, by definition, not me.
- **The Wazuh server lives at home, not on the VPS.** Keeping the SIEM off the exposed host means an attacker who compromises the honeypot never has line of sight to the log store or the detections written against them. The VPS is treated as untrusted.
- **WireGuard is dialled out from home to the VPS.** The home connection is behind Starlink CGNAT with no inbound reachability, so the tunnel is established from the home side; the VPS acts as the fixed public endpoint the agent traffic rides back through.
- **`sshd` is ordered after the tunnel at boot** (a systemd drop-in requiring `wg-quick@wg0`), so a reboot can't leave admin SSH unable to bind its tunnel-only address.
- **The indexer heap is capped** so the Wazuh stack coexists with an existing local LLM inference workload on the same home machine.

## Components

| Component | Role | Host |
|-----------|------|------|
| Cowrie | Medium-interaction SSH honeypot; logs credentials tried and commands run | VPS |
| Wazuh agent | Ships Cowrie JSON and host telemetry to the manager | VPS |
| WireGuard | Encrypted transport between VPS and home lab | Both |
| Wazuh manager | Rule engine, decoders, alerting | Home lab |
| Wazuh indexer + dashboard | Storage and visualisation | Home lab |
| Test VM (Oracle Linux 9) | Disposable purple-team host; Wazuh agent + Atomic Red Team | Home lab |

## Detection engineering

The point of the lab is the detections, not the honeypot itself. Coverage, mapped to MITRE ATT&CK (see [`wazuh/rules/local_rules.xml`](wazuh/rules/local_rules.xml)):

- **Credential brute force (T1110)** — a single failed login is low-severity raw material; 6+ from one source within 120s raises a high-severity burst alert, grouped per source IP.
- **Valid-account use (T1078)** — a "successful" login into the Cowrie sandbox (an attacker past the deliberately weak credentials) is the headline event, level 12.
- **Post-login command capture (T1059)** — every command typed in the fake shell is decoded and surfaced, revealing the common patterns (recon, payload downloads, persistence).
- **Ingress tool transfer (T1105)** — file downloads pulled into the sandbox are flagged.
- **Native JSON decoding** — Cowrie's JSON is parsed by Wazuh's built-in `json` decoder, so rules match fields (`eventid`, `src_ip`, `username`, `input`) directly. No custom decoder ([why](wazuh/decoders/README.md)).
- **Purple-team validation (in progress)** — a disposable Oracle Linux 9 VM runs a Wazuh agent and [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team); scripted ATT&CK techniques are executed there and each is confirmed to fire the expected alert, pairing *technique executed → alert raised*. Atomics run on the monitored VM (host-based detections), never on the honeypot or the manager.

### Coverage

Rules live in [`wazuh/rules/local_rules.xml`](wazuh/rules/local_rules.xml). Validated against both self-tests and live scanner traffic hitting the exposed honeypot.

| Rule | Detection | ATT&CK | Severity | Validated |
|------|-----------|--------|----------|-----------|
| 100100 | Failed SSH login to honeypot | T1110 — Brute Force | 5 | ✅ live traffic |
| 100101 | Brute-force burst (≥6 failures / 120s, per source) | T1110 — Brute Force | 10 | ✅ live traffic |
| 100102 | Successful login into the sandbox | T1078 — Valid Accounts | 12 | ✅ `wazuh-logtest` |
| 100103 | Command executed in the fake shell | T1059 — Command & Scripting Interpreter | 6 | 🚧 deployed |
| 100104 | File download pulled into session | T1105 — Ingress Tool Transfer | 10 | 🚧 deployed |

Wazuh auto-enriches each alert from the `<mitre>` tag — e.g. rule 100102 surfaces as *Valid Accounts* across the Defense Evasion, Persistence, Privilege Escalation, and Initial Access tactics in the ATT&CK dashboard.

## Build progress

- [x] Provision VPS (Ubuntu 24.04 LTS)
- [x] Host hardening — real SSH moved to tunnel-only, key auth, boot ordering
- [x] WireGuard tunnel: VPS ↔ home lab
- [x] Wazuh server (manager + indexer + dashboard) on home lab
- [x] Enrol Wazuh agents (VPS + home hosts)
- [x] Deploy Cowrie on the VPS (dual-stack listener, public 22 redirected)
- [x] Ship Cowrie logs into Wazuh (native JSON, agent → manager verified)
- [x] Custom detection rules + MITRE ATT&CK mapping (T1110 / T1078 validated live)
- [x] Stand up disposable purple-team VM (Oracle Linux 9, Wazuh agent enrolled)
- [ ] Atomic Red Team purple-team validation *(in progress)*
- [ ] Telegram alerting on high-severity events
- [ ] Publish 30-day honeypot analysis

## Repository layout

```
wazuh/
  rules/          custom detection rules (local_rules.xml)
  decoders/       note on why native JSON decoding is used
cowrie/           honeypot config (sanitised — overrides only)
wireguard/        tunnel config templates (no keys)
docs/             architecture notes, build log, analysis
```

## Notes on safety and scope

This is a personal research lab. The honeypot is a sandbox by design and is not connected to any production system. Configuration published here is sanitised — no keys, tokens, or real IPs. Attacker source data shown in any analysis is aggregated and used only to characterise attack patterns.

---

*Part of an ongoing home-lab portfolio. Companion repos: [t2-macbook-linux](https://github.com/LVZFR), [home-network-monitoring](https://github.com/LVZFR).*
