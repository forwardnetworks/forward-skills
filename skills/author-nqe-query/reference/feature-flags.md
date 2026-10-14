# A field behind an NQE feature flag

Read when a query uses `device.securityPolicy`, `device.services`, host ports, `cisFindings`, LLDP chassis ids, or any field that lint or Forward reports as "Record does not have field" although the name is right, and when an organization's data model differs from the documented one.

Forward serves each organization a data model filtered by its **feature flags** (organization properties). A field behind a flag that is off does not exist for that organization, so a correct query fails there with "Record does not have field". The flags come from Forward's own schema filter (read in Forward's source, build 261014); they are experimental or recently launched, and Forward plans to retire each once it is on for everyone.

| `fwdctl` name | Forward org property | Default | What it exposes | Who can change it |
|---|---|---|---|---|
| `security-rules-panos` | `NQE_SECURITY_RULES_PANOS` | off | `device.securityPolicy` for Palo Alto PAN-OS: scopes, address, service and application objects and groups, rulebases and rules | on-premises org admin, otherwise Forward |
| `security-rules-fortios` | `NQE_SECURITY_RULES_FORTIOS` | off | the same `device.securityPolicy` for Fortinet FortiOS | same |
| `cis-benchmarks` | `CIS_BENCHMARKS` | on | `network.cisDatabase`, `device.cisFindings` | same |
| `additional-host-info` | `NQE_ADDITIONAL_HOST_INFO` | off | extra host fields: operating system, addresses, gateway interfaces, security zones, listening ports | Forward only |
| `peer-lldp-cdp-capability` | `PEER_LLDP_CDP_CAPABILITY` | on | the capabilities a discovery neighbor advertises | on-premises org admin, otherwise Forward |
| `lldp-chassis-id` | `NQE_LLDP_CHASSIS_ID` | off | the LLDP chassis id of a neighbor | same |
| `device-services` | `NQE_DEVICE_SERVICES` | off | `device.services`: DNS, log, NTP and TACACS server configuration | same |
| `trim-builtin` | `NQE_TRIM_BUILTIN` | on | the `trim()` function | same |
| `parquet` | `NQE_PARQUET` | off | how Forward stores results: no effect on what you can write | same |
| `numeric-column-summaries` | `NQE_DASHBOARD_PANELS` | on | summary statistics of numeric result columns: no effect on what you can write | same |

The security policy model is experimental for Palo Alto and Fortinet: expect it to change. A model needs data, so a flag turned on after a snapshot was processed may not give that snapshot the new data; the property also controls whether Forward parses the rules (read in Forward's parsing configuration; not confirmed on a live organization), so test against a snapshot processed after it was enabled.

## What to do

1. **Read what lint says.** `fwdctl nqe lint` puts a `hint` on the unknown-field diagnostic. When the field is behind a flag the hint names the org property and the `fwdctl` feature name.
2. **Check offline as the organization would.** `fwdctl nqe features` lists the flags and which are on; `fwdctl nqe lint --features security-rules-fortios q.nqe` (names or org property names, comma list; `FORWARD_NQE_FEATURES` too) checks as an organization with that flag on. This tells you the query is valid for such an organization, not that yours has the flag.
3. **Find out whether the organization has it.** The authority is the organization's own data model: `fwdctl nqe lint --org q.nqe` (needs a login), or `fwdctl context schema "<term>"` with a login, which searches the live model. A field that is absent there is off for that organization. `inspect-platform` area `org_properties` shows the property's `effective` value, but needs Forward support permission.
4. **If it is off, do not work around it.** The field does not exist for that organization. Say so, name the property, and who can change it (the table). Changing it is `edit-org-property` (a dry run first; it needs the same authority). These properties are classified `caution`: turning one off breaks saved queries that use its fields, and some change what is parsed on the next processing, so the change is the organization's decision. To try a flag without leaving it on, use the skill's guarded window: it sets the value, reprocesses ONE snapshot, waits, and always puts the original value back.
5. **Do not report a missing field as a typo.** `fwdctl nqe lint` also suggests the closest real field name; a flagged field has no near neighbor, and its hint says why.

`fwdctl context schema` without `--features` and without a login shows Forward's full model, flagged fields included: finding a field there does not mean the organization can use it.

## Experimental lint rules

`fwdctl nqe features` also lists fwdctl's own experimental lint rules (`--experimental NAME`, or `FORWARD_NQE_EXPERIMENTAL`, `all`). They only warn, are off by default, and each states what it rests on. Today: `perf-command-response` (see `reference/gotchas.md`, Speed).
