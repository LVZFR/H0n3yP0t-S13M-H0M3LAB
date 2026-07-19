# Decoders

**No custom decoder is needed for Cowrie on this stack.**

The VPS Wazuh agent reads Cowrie's log with `<log_format>json</log_format>`, so
Wazuh's built-in `json` decoder parses every line into fields (`eventid`,
`src_ip`, `username`, `password`, `input`, …). The rules in
`../rules/local_rules.xml` match on those fields directly via
`<decoded_as>json</decoded_as>`.

An earlier design used a custom decoder with `<prematch>^{"eventid":</prematch>`.
That was dropped: real Cowrie log lines begin with `{"session":...`, so the
prematch anchored on `eventid` never matched. Native JSON decoding avoids the
problem entirely and is order-independent.

## Agent-side config (on the VPS, inside `<ossec_config>`)

```xml
<localfile>
  <log_format>json</log_format>
  <location>/home/cowrie/cowrie/var/log/cowrie/cowrie.json</location>
</localfile>
```
