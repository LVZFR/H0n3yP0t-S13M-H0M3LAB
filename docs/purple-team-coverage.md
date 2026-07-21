# Purple-Team Validation — Coverage Report

**Method:** ATT&CK techniques executed deliberately on a monitored host, then verified
to raise the expected alert in Wazuh. This is the difference between *"I wrote a
detection"* and *"I attacked my own lab and watched it get caught."*

**Test host:** disposable Oracle Linux 9 VM (`ol9-client`, agent 002) on the home LAN,
running a Wazuh agent and [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team).
Attacks are executed **on the VM**, never on the honeypot VPS or the manager — running
them on the honeypot would contaminate the live attacker dataset with self-generated noise.

**Scope note:** Atomic Red Team exercises *host-based* detections on the monitored VM.
The Cowrie honeypot rules (100100–100104) are validated separately by attacking the
honeypot directly — see [`../wazuh/rules/local_rules.xml`](../wazuh/rules/local_rules.xml).

---

## Coverage

| Technique | ATT&CK ID | Tactic | Executed | Detected | Rule fired |
|-----------|-----------|--------|----------|----------|------------|
| Create local account | T1136.001 | Persistence | `useradd evil_user` via Atomic Red Team | ✅ | **5902** — "New user added to the system" (level 8) |
| SSH brute force | T1110 | Credential Access | 8 failed SSH auths as a non-existent user | ✅ | **5710** ×7 — "Attempt to login using a non-existent user" (level 5)<br>**5712** — "brute force trying to get access to the system" (level 10) |
| Clear command history | T1070.003 | Defense Evasion | — | 🚧 not yet tested | — |

### What the T1110 result demonstrates

The individual failures land at level 5, and Wazuh's built-in correlation rule escalates
the burst to level 10. That is structurally the same design as the custom Cowrie rules
written in Phase 7 (100100 single failure → 100101 burst at level 10), which is a useful
cross-check: the hand-written honeypot detection follows the same pattern the vendor
ruleset uses for host-based brute force.

---

## Incidental detections

Not planned tests, but observed during the exercise and worth recording — they answer
real operational questions:

| Event | Rule | Level | Why it matters |
|-------|------|-------|----------------|
| `sudo` to root | 5402 | 3 | Privilege escalation on a monitored host is logged |
| Wazuh agent disconnected | 504 | 3 | The manager alerts when an endpoint goes silent — i.e. there *is* a detection for "a monitored host went dark" |
| Listening ports changed | 533 | 7 | New/closed listeners are surfaced (useful for backdoor/persistence detection) |

The **rule 504** finding came out of a real incident during the build: the test VM's
virtual NIC dropped, and the agent silently stopped reporting. From the manager it
appeared merely "disconnected" with no indication of cause. Worth knowing that the
silence itself is alertable — an endpoint that stops reporting is exactly what a
compromised host might look like.

---

## Gaps and next steps

- **T1070.003 (Clear Command History)** not yet executed. If it produces no alert, that
  is a legitimate coverage gap to document and close with a custom rule rather than a
  failure to hide.
- Further techniques worth adding to widen coverage: **T1053.003** (cron persistence),
  **T1548.003** (sudoers modification), **T1087.001** (account discovery).
- Honest reporting principle: this table records **misses as well as hits**. A documented
  gap plus a proposed rule to close it demonstrates more detection-engineering judgement
  than an all-green table.
