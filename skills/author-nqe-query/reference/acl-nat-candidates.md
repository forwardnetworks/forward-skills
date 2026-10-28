# Which NAT entries can apply to a security rule (candidates, not proof)

Forward's model has no link between a security rule and a NAT rule (the policy has no NAT field), so it is derived by comparing `device.aclEntries` with `device.natEntries`. The result is a list of candidates: it says a NAT entry could match traffic a rule matches, not that it does. From a field session on PAN-OS (Forward 26.10); `nqe lint --org` passed, and it has not been run by this repository's tests. The egress step is the least checked.

The steps, in order (each narrows the pairs):
1. Pre-NAT source and destination both overlap. A security rule matches the addresses before NAT.
2. Ingress interfaces overlap (on PAN-OS the rule's from-zone).
3. Egress interfaces overlap, only when the NAT has no destination rewrite (a rule's to-zone is the zone after NAT).
4. IP protocol and destination port ranges overlap.

```nqe
rangesOverlap(a, b) = length(a) == 0 || length(b) == 0 || length(foreach x in a foreach y in b where x.start <= y.end && y.start <= x.end select 1) > 0;
ifacesOverlap(a, b) = length(a) == 0 || length(b) == 0 || length(foreach x in a foreach y in b where x == y select 1) > 0;
@query
scoreNat(Device: String) =
foreach device in network.devices
where device.name == Device
let nats = (foreach nat in device.natEntries select { nat: nat, srcSet: ipAddressSet(nat.headerMatches.ipv4Src), dstSet: ipAddressSet(nat.headerMatches.ipv4Dst), dnat: length(foreach r in nat.rewrites where length(r.ipv4Dst) > 0 select 1) > 0 })
foreach acl in device.aclEntries
let aSrc = ipAddressSet(acl.headerMatches.ipv4Src)
let aDst = ipAddressSet(acl.headerMatches.ipv4Dst)
let pairs = (foreach n in nats
  where !isEmpty(intersect(aSrc, n.srcSet)) && !isEmpty(intersect(aDst, n.dstSet))
  let ing = ifacesOverlap(acl.metadataMatches.ingressInterfaces, n.nat.metadataMatches.ingressInterfaces)
  let eg = n.dnat || ifacesOverlap(acl.metadataMatches.egressInterfaces, n.nat.metadataMatches.egressInterfaces)
  let svc = rangesOverlap(acl.headerMatches.ipProtocol, n.nat.headerMatches.ipProtocol) && rangesOverlap(acl.headerMatches.tpDst, n.nat.headerMatches.tpDst)
  select { name: n.nat.name, ing: ing, eg: eg, svc: svc, dnat: n.dnat })
select { rule: acl.name, overlap: length(pairs), plusIngress: length(foreach p in pairs where p.ing select 1), plusEgress: length(foreach p in pairs where p.ing && p.eg select 1), plusService: length(foreach p in pairs where p.ing && p.eg && p.svc select 1) };
```

Run it per device (the device name is the parameter). Cost grows with entries squared: about a minute for a few thousand NAT and ACL entries on one PAN-OS virtual system.

Things to know:
- "any" in an address field is 0.0.0.0/0, not an empty list; an empty interface or port list is treated as any.
- `device.securityZones` can be empty, so compare interfaces, not zones.
- `aclEntries` include entries Forward generates that are not in the configuration.
- A rule that overlaps every NAT entry is common (a broad rule); read the counts per step, not only the first.
- Say in the answer that these are candidates, derived and not proven by Forward.
