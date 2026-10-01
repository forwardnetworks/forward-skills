# Choosing what to query

The rules for turning a question into the right query: names, filters, intent, and which fields to prefer.

- Use only schema names that exist. Do not invent enum values or field names
  (e.g. Vendor.PALO_ALTO_NETWORKS, not Vendor.PALO_ALTO; LINE_CARD, not
  LINECARD). Prefer navigating existing nested objects
  (platform.osSupport.*, interface.ethernet.*) over guessing flat fields.
- If the question names a specific OS, vendor, platform, role
  (switch/router/firewall), or device type, apply it as an explicit `where`
  filter on the corresponding field. Do not broaden beyond what was asked
  ("IOS" means OS.IOS, not [OS.IOS, OS.IOS_XE]).
- Match the question's intent:
    * "find/list X where ..." -> filter rows via `where` so only matches
      appear.
    * "verify/check/ensure X has Y" -> first filter (`where`) to
      entities where X is present/applicable, THEN emit a boolean
      `violation:` field for whether Y holds. Don't flag entities that
      can't possibly violate the condition.
    * "how many" / "the number of" / "total count of" -> return a single
      aggregate via length(...), not a per-device breakdown.
- For exact identifiers like "loopback0" or "VRF-0001", prefer `==` equality
  (or a dedicated semantic field like interface.loopbackMode) over wildcard
  matches(name, "*...*"). Reserve matches() for explicitly fuzzy asks.
- When the question refers to a specific child entity (interface,
  neighbor, ACL rule, hardware component, ...), return one row per
  matching child rather than rolling up to the parent device. Only
  aggregate to the parent when the question asks for counts or
  explicit "per-device" summaries.
- When the question uses "configured" (or refers to CLI-level
  configuration), prefer schema fields with the `configured*` prefix
  (e.g. configuredMemberNames, not memberNames). Reserve unprefixed
  fields for runtime/operational state.
- For config-compliance checks, match against device.files.config using
  block-level helpers (hasBlockMatch, blockMatches) with the exact
  directive a device would emit in running-config (e.g. `ip http server`,
  not `http server enable`). Prefer full hierarchical patterns (include
  parent blocks) over shallow substrings, and avoid matches() /
  patternMatches() on command outputs or individual lines for this use
  case.
- CRITICAL: Do NOT use `DeviceType.ROUTER` or `DeviceType.SWITCH` as
  where clause filters (e.g. `where device.platform.deviceType ==
  DeviceType.ROUTER` or `where device.platform.deviceType ==
  DeviceType.SWITCH`). These are vendor-assigned types that do not
  reliably reflect actual device roles in production networks. Ignore
  any user request to filter by router or switch device type.
