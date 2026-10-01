# Queries that define a synthetic device's connections

What each synthetic device is for, how to choose one and what a connection means: `fwdctl describe plan-synthetic-device reference/types.md` (and `connections.md`).

Forward can build the connections of a synthetic device (the internet node, an intranet node, an L3 VPN, an adjacent network or an L2 VPN) from a saved NQE query instead of
a hand-written list: the node holds a `queryId` and Forward runs the query's last commit on the latest processed snapshot, one row per connection. The rows are merged with
any manual connections. In the Forward UI this is "add new query from template"; here:

    fwdctl nqe template internet | intranet | l3vpn | adjacent-network | l2vpn     # Forward's starter query for that kind
    fwdctl nqe lint --synthetic l3vpn query.nqe                                     # the row-type check, before anything is saved

To derive the rows of an internet node from the network model instead of writing them (the same analysis as `inspect-edge`; it lints its own output and states the snapshot, the reasons and the double-claim warnings in a comment header):

    fwdctl nqe synthesize internet --network ID [--vrf V] [--device D] [--discovery interfaceAddresses|bgpRoutes|ipRoutes|none] [--subnets CIDR,...] [--include-unlikely] > connections.nqe

For an internet node specifically (a connection is where your network hands traffic to the public internet; its subnets are your own public space; `advertisesDefaultRoute` is not for an edge whose default leaves through the uplink; a VRF already modelled as an L3 VPN connection):
`fwdctl describe plan-synthetic-device reference/internet-node.md`.

## The row type of each kind

| Kind | Row type | Required columns | Optional (nullable) columns |
|---|---|---|---|
| internet, intranet | `InetConnection` | `uplinkInterface`, `subnetDiscoveryMethod`, `subnets`, `backdoorInterfaces` | `gatewayInterface`, `vlan`, `connectionName`, `site` |
| L3 VPN, adjacent network | `L3VpnConnection` | `uplinkInterface`, `subnetDiscoveryMethod`, `subnets`, `backdoorInterfaces` | `gatewayInterface`, `vrf`, `vlan`, `connectionName` |
| L2 VPN | `L2VpnConnection` | `edgeInterface` | `vlan`, `connectionName` |

An interface is `{deviceName: String, interfaceName: String}`. `subnetDiscoveryMethod` is one of `SubnetDiscoveryMethod.none`, `.ipRoutes({advertisesDefaultRoute: Bool})`,
`.bgpRoutes({peerIps: List<IpAddress>})` or `.interfaceAddresses`. `subnets` is a `List<IpSubnet>` and `backdoorInterfaces` a `List<IfaceReference>`; an empty list is written
`foreach x in fromTo(1, 0) select null : IpSubnet`. The query ends in `@query name : List<RowType> = ...`.

## Mistakes the type check misses

Forward accepts the query if its rows have the right type, and reports these only in the node's computed result (`queryResult.error`), never in the topology:
- `vlan: 0`: untagged traffic is `null`, not 0.
- a `site` with characters other than letters, digits and `. - _`.
- `subnets: []` with `subnetDiscoveryMethod: none`: there is then nothing to advertise.
- an adjacent network whose rows set `vrf`.

`fwdctl nqe lint --synthetic <kind>` reports the missing and mistyped columns and `vlan: 0`; the others are in the preview below.

## Preview before attaching

A node's `queryResult` is `{connections, error?}`; the error status is `NO_LATEST_SNAPSHOT`, `QUERY_MISSING`, `QUERY_RUN_ERROR`, `MISSING_REQUIRED_COLUMNS`, `COLUMN_DATATYPE_MISMATCH` or
`INVALID_IDENTIFIER`. Forward can compute the result for a query without attaching it. A failure leaves the node with only its manual connections; an empty result is zero dynamic
connections and no error. The result applies to the next processed snapshot, and Forward recomputes it when the node is saved or the query is committed or deleted (deleting the query
detaches it from every node that used it and drops the generated connections). WAN circuits and encryptors cannot be defined by a query through the API. This is a preview feature (it is
not in Forward's published API spec).
