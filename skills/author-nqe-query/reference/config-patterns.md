# Reading configuration from NQE (BGP policy, static routes, block patterns)

Read when the answer is in the device's configuration text and not in the model: which route-map or prefix-list a BGP neighbor uses,
configured static routes, or any block-pattern query that returns nothing.

## Contents

- What the model does not hold
- Block patterns that were run (BGP policy by vendor, measured forms)
- Worked example: every NX-OS BGP route-map form, unioned
- Why a pattern returns nothing, or will not compile
- Configured static routes (starting patterns)
- Endpoints: which profile asks for what, and which product it is

## What the model does not hold

- **BGP policy per neighbor.** `BgpNeighbor` has `neighborAddress`, `peerDeviceName`, `peerVrf`, `peerRouterId`, `sessionState`, `enabled`,
  `description`, `peerAS`, `localAS`, `peerType` and `statistics`. No route-map, prefix-list or route-policy name. `bgpRib` carries the
  effect of policy (`adjRibInPost`, `adjRibOutPost`), not its name. The names come from `device.files.config`.
- **Configured static routes.** On the one network a peer session measured, routing protocol identifiers held BGP and OSPF, not STATIC (unverified elsewhere). Static routes appear
  only as installed forwarding entries (`nextHops[*].originProtocol == OriginProtocol.STATIC`), which says what is in the table, not what
  is configured (prefix, next hop, distance, tag, name). Configured statics are `ip route` lines (or the vendor's form) in `device.files.config`.

## Block patterns that were run

Match with `blockMatches(device.files.config, pattern)` and read `match.data`. Indentation in the pattern is the nesting of the config.

NX-OS, neighbors inside a VRF block (matched on a live NX-OS network; the same pattern without the `vrf` level matched nothing there):

    nxvrf = ```
    router bgp {asn:string}
      vrf {vrf:string}
        neighbor {peer:string}
          address-family {afi:string} {safi:string}
            route-map {policy:string} {direction:string}
    ```;

IOS, IOS-XE and Arista, the flat form:

    flat = ```
    router bgp {asn:string}
      neighbor {peer:string} route-map {policy:string} {direction:string}
    ```;

A peer investigation measured these forms on one large multi-vendor network (row counts of `blockMatches`, all patterns lint clean; counts
differ per network, so use them to see which forms matter, not as expected values):

| Vendor | Form | Rows |
|---|---|---|
| NX-OS | `router bgp > vrf > neighbor > address-family > route-map` | 836 |
| NX-OS | the same without the `vrf` level (default VRF) | 603 |
| NX-OS | `router bgp > neighbor > inherit peer {tmpl}` (default VRF) / under `vrf` | 1,624 / 834 |
| NX-OS | `router bgp > template peer {tmpl} > address-family > route-map` | 130 |
| NX-OS | prefix-list under a neighbor address family / under a template | 0 / 0 |
| Arista | flat `neighbor X route-map P in\|out` / under `vrf` | 982 / 45 |
| Arista | peer-group members `neighbor X peer-group G` / declared `neighbor G peer-group` | 44 / 63 |
| IOS-XE | `address-family {afi} > neighbor X route-map P in\|out` (prefix-list variant) | 306 (53) |
| IOS-XE | `address-family {afi} {kw} {vrf}` (the `vrf NAME` form) | 86 |
| IOS-XE | flat route-map / prefix-list | 12 / 20 |
| IOS-XE | peer-group members / declared groups | 186 / 264 |

What it shows: on NX-OS most policy arrives by **template inheritance**, so a correct per-neighbor answer joins `neighbor.tmpl` to
`template.tmpl` on the same device; the neighbor form that carries policy is the two-line form (`neighbor X`, then a child `remote-as N`),
and the one-line `neighbor X remote-as N` matched nothing there. On IOS-XE the policy sits under `address-family`, not in the flat form.
On Arista and IOS-XE peer-group members inherit from the group. IOS-XR was not measured. To see which forms a network uses, run the union
of the forms (`blockMatches(c, a) + blockMatches(c, b)`) with `fwdctl nqe run --count-by FIELD`.

**Raw text misleads on NX-OS fabric configs.** `configuration.txt` also holds nested `configure profile ...` template blocks that contain
`router bgp ... neighbor ... route-map ...` at deeper indentation. A raw search (`inspect-device-files`) finds route-maps that are not
live neighbor configuration; the parsed tree that `blockMatches` reads ignores them. That is why a pattern can return nothing on a device
whose raw text shows route-map lines.

### Worked example: every NX-OS BGP route-map form, unioned

One query that returns one row per attachment for all the forms measured above (a neighbor inside a `vrf` block or in the default VRF, policy
directly on the neighbor's address family, and policy by `inherit peer` to a `template peer`). Run it first with `fwdctl nqe run --network ID
--file F --count-by form` to see which forms a network uses. The inherit rows carry the template name in `policy`: a per-neighbor answer joins
them to the "template policy" rows on device and template name. The one-line `neighbor X remote-as N` forms are left out on purpose (they
matched nothing; they occur only in nested configure-profile blocks). It lints with one Bag-ordering warning per `blockMatches` and ran on a
live network, where it returned the four forms that network has (default inherit, vrf inherit, template policy, default direct). Static routes
and IOS-XR are not in it.

    vrfDirect = ```
    router bgp {asn:string}
      vrf {vrf:string}
        neighbor {peer:string}
          address-family {afi:string} {safi:string}
            route-map {policy:string} {direction:string}
    ```;
    dflDirect = ```
    router bgp {asn:string}
      neighbor {peer:string}
        address-family {afi:string} {safi:string}
          route-map {policy:string} {direction:string}
    ```;
    vrfInherit = ```
    router bgp {asn:string}
      vrf {vrf:string}
        neighbor {peer:string}
          inherit peer {tmpl:string}
    ```;
    dflInherit = ```
    router bgp {asn:string}
      neighbor {peer:string}
        inherit peer {tmpl:string}
    ```;
    tmplPolicy = ```
    router bgp {asn:string}
      template peer {tmpl:string}
        address-family {afi:string} {safi:string}
          route-map {policy:string} {direction:string}
    ```;

    foreach device in network.devices
    where device.platform.os == OS.NXOS
    let cfg = device.files.config
    let rows =
      (foreach m in blockMatches(cfg, vrfDirect)  select {form: "vrf direct",      vrf: m.data.vrf,   peer: m.data.peer, afi: m.data.afi, policy: m.data.policy, direction: m.data.direction})
      + (foreach m in blockMatches(cfg, dflDirect)  select {form: "default direct",  vrf: "default",    peer: m.data.peer, afi: m.data.afi, policy: m.data.policy, direction: m.data.direction})
      + (foreach m in blockMatches(cfg, vrfInherit) select {form: "vrf inherit",     vrf: m.data.vrf,   peer: m.data.peer, afi: "",         policy: m.data.tmpl,   direction: "template"})
      + (foreach m in blockMatches(cfg, dflInherit) select {form: "default inherit", vrf: "default",    peer: m.data.peer, afi: "",         policy: m.data.tmpl,   direction: "template"})
      + (foreach m in blockMatches(cfg, tmplPolicy) select {form: "template policy", vrf: "",           peer: m.data.tmpl, afi: m.data.afi, policy: m.data.policy, direction: m.data.direction})
    foreach r in rows
    select {device: device.name, form: r.form, vrf: r.vrf, peer: r.peer, afi: r.afi, policy: r.policy, direction: r.direction}

## Why a pattern returns nothing, or will not compile

- **Capture types.** `{x:ip}` does not compile ("no viable alternative"). Use `string`, `number` or the address types `ipv4Address`
  (`{peer:ipv4Address}`). Capture as `string` and compare when unsure.
- **A pattern is literal about the line.** A neighbor line that carries `remote-as` in the pattern silently skips devices that write
  `neighbor X` and then a child `remote-as N` on the next line (NX-OS often does). There is no error. Write one pattern per form and
  add the results: `blockMatches(c, a) + blockMatches(c, b)`.
- **Peer groups.** On IOS-XE and Arista many "neighbors" are peer-group names, and a peer inherits policy from its group. A flat pattern
  returns the group's line, not each member's. Resolve membership (`neighbor X peer-group G`) with a second pattern and join on the name.
- **IOS-XR** nests `route-policy` per neighbor and address family; it needs its own pattern.
- **Bag warning.** `blockMatches(device.files.config, p)` lints with "first argument is a Bag, which has non-deterministic ordering".
  The query still runs and the rows are right; to silence it, order the config first. The warning will become an error in a later release.
- **No rows is not a result.** Zero matches can mean no such config, or a pattern that does not fit the form. Test the pattern on one
  device whose config you have read (`inspect-device-files`, search) before trusting an empty answer, and say which forms you covered.

## Configured static routes (starting patterns)

Installed static routes (`nextHops[*].originProtocol == OriginProtocol.STATIC` in the forwarding table) are not configured routes: they
carry no distance, tag or name, they include whatever the platform installs (a measured NX-OS fabric produced over a million), and a
query over them can hit NQE's 1,048,576-row cap. Read configured statics from the config text. Counts below are matching lines on one
multi-vendor network (IOS, IOS-XE, NX-OS; no Arista there), from a live run; the patterns capture prefix and next hop only, not
distance, tag, `name` or permanent:

    ios      = ```ip route {prefix:string} {mask:string} {nh:string}```;
    iosvrf   = ```ip route vrf {vrf:string} {prefix:string} {mask:string} {nh:string}```;
    nxosvrf  = ```
    vrf context {vrf:string}
      ip route {prefix:string} {nh:string}
    ```;
    nxosglob = ```ip route {prefix:string} {nh:string}```;

IOS and IOS-XE: the global pattern's `{prefix}` is also `vrf` on `ip route vrf ...` lines, so add `where match.data.prefix != "vrf"` (not
yet verified) and read the VRF lines with the second pattern. NX-OS: global statics are top-level `ip route`, VRF statics are nested
under `vrf context`; the nested form returned the large majority of the NX-OS lines. Arista uses `ip route vrf NAME ...`; untested here.
Report which forms were covered: a vendor or form with no pattern is missing from the answer, not empty.

## Endpoints: which profile asks for what, and which product is it

`network.endpoints` has no vendor or model field: only `name`, `locationName`, `tagNames`, `snapshotInfo`, `profileName`, `httpResults`,
`cliCommandResponses` and `snmpOutputs`. A peer investigation ran these two queries on a live network with SNMP endpoints (they lint
clean here; not reproduced here, the networks available had no endpoints). On that network `cliCommandResponses` and `httpResults` were
empty for every endpoint, and each profile asked only for MIB-II system, interfaces, ifXTable and address tables plus a vendor serial and
version OID, so no configuration reached the model.

    // which OIDs each profile requests
    foreach e in network.endpoints
    foreach o in e.snmpOutputs
    select distinct {profile: e.profileName, alias: o.alias, oid: o.requestedOid}

    // which product each endpoint is (group client-side by profile, alias, value)
    foreach e in network.endpoints
    foreach o in e.snmpOutputs
    where o.alias == "sysObjectId" || o.alias == "sysDescr"
    foreach r in o.rawOidEntries
    select {profile: e.profileName, alias: o.alias, value: r.rawValue}

`rawOidEntries` has `oid`, `oidNumbers` and `rawValue` (not `value`). `sysDescr` groups by product family and `sysObjectId` by enterprise
subtree. Forward collects only the OIDs a profile lists, so "does this device expose OID X" cannot be answered from the model.
