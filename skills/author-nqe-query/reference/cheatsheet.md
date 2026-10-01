# NQE language cheat sheet

Syntax forms and the roots of the data model. Look up real field names with `fwdctl context schema "<term>"`.

- Iterate:       foreach <x> in <collection>
- Filter:        where <bool-expr>
- Bind:          let <x> = <expr>
- Project:       select { Field1: expr1, Field2: expr2 }
                 (or `select distinct expr`)
- Inline table:  [{ f: v, ... }, { f: v, ... }]
- Tagged union:  when <x> is tag(v) -> v; otherwise -> default
- Parameterize:  @query
                 query(numberOfDays: Number) = foreach device in network.devices ...
- Imports:       import "@fwd/L3/IpAddressUtils";
- Null-safe:     a?.b   isPresent(x)   (x ?: fallback)

Common schema roots:
- network.devices         device.platform, .interfaces, .networkInstances,
                          .aclEntries, .hosts, .cveFindings, .files, .outputs,
                          .snapshotInfo, .system, .bgpRib, .stp, .ha, .natEntries
- network.cloudAccounts   .vpcs -> .subnets, .securityGroups, .routeTables,
                          .computeInstances
- network.locations
