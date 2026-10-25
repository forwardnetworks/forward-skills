---
name: forward-mcp
description: "Answers questions from Forward's digital twin via its official MCP server: paths, devices, VLANs, VRFs, hosts, CVEs, NQE. Use when connected to <host>/api/mcp; every call needs networkId."
license: Proprietary
---

# Forward MCP

Unofficial guidance for an agent connected to Forward Networks' official MCP server, served by Forward itself at `<forward-host>/api/mcp` (streamable HTTP; SaaS and self-hosted). It is not an official Forward document. It reads a snapshot of the network model; it never touches a device and it cannot change Forward. How to connect: `reference/connecting.md`.

## Read-only
Every tool Forward exposes over MCP is read-only. None creates, changes or deletes anything, and none collects, reprocesses or deletes snapshots. If the user asks for a change, say Forward's MCP cannot make it and stop. The token belongs to a real Forward user, so the MCP reaches only the networks and permissions that user has: a tool can be listed and still be denied for one network.

## Always supply a networkId
An MCP request carries no network context. **Every call needs `networkId`** and may take `snapshotId` (omitted means the network's latest processed snapshot). **No tool lists networks or snapshots**: if the user has not given a network ID, ask for it; do not guess or iterate over IDs. Prefer an explicit `snapshotId` when the user needs a repeatable answer or a before and after, and say which snapshot the answer came from.

## Tools
`tools/list` is the source of truth: Forward can add or change tools, and the list is narrowed by license and by whether the organization has Forward AI enabled. Read the input schema there before the first call to a tool; do not guess argument names.

| User intent | Tool | Arguments (besides `networkId`, `snapshotId`) |
|---|---|---|
| Can A reach B? Where does it fail? | `get_paths` | `dstIp` (required) plus `srcIp` or `fromDevice` (one is required, `fromDevice` is a device name); optional `srcPort`, `dstPort` (strings, ranges such as `8080-8088` allowed), `ipProto` (6 TCP, 17 UDP, 1 ICMP). Needs a path-viewing permission |
| About device X | `get_device` | `deviceName` |
| About interface Y | `get_interface` | `deviceName`, `interfaceName` |
| What is on VLAN N | `get_vlan` | `vlanId` (integer) |
| What is in VRF V | `get_vrf` | `vrfName` |
| Where is host H | `get_host` | `hostSpecifier` (name, cloud host ID, IP or MAC) |
| What is this cloud resource | `get_cloud_object` | `objectId` |
| Where does subnet S live | `get_subnet_locations` | `subnetAddress` (for example `10.0.0.0/24`) |
| Which CVEs were checked on device X | `get_device_vulnerabilities` | `deviceName` |
| What is CVE-... | `get_vulnerability` | `cveId` |
| How big is this network | `get_network_complexity` | none; exact counts of devices (by vendor, type, model, OS version), VLANs, VRFs, ACLs, links, cloud objects |
| Ask Forward a data question | `nqe` | `prompt`: a complete, self-contained request. Forward writes and runs the query (needs an NQE permission and AI) |
| Write an NQE query | `generate_nqe_query` | `prompt`; returns a query without running it (needs an NQE permission and AI) |
| How do I do X in NQE | `search_docs` | `query`: a self-contained search phrase; returns at most 2 documentation chunks (AI) |

The permission names and the row and chunk caps above come from the tool descriptions and from Forward's MCP page, which was not publicly available when this was written: treat them as reported, not guaranteed. If `nqe` is absent from `tools/list`, NQE is unavailable for this org: say so, do not work around it.

## Working rules
1. **Start narrow.** One call answers one object or one path. Chain calls for a bigger question (a path, then the devices on it, then their CVEs).
2. **`get_paths` returns one path result**, not every possible path. It carries `forwardingOutcome`, `securityOutcome`, the `hops`, `totalHits` (can be a lower bound), `timedOut`, and a `queryUrl` that opens the same trace in Forward. State the source, destination and snapshot, and give the link.
3. **`nqe` is generated, so check `status`.** The result has `query`, `summary`, `status` and, on success, an `executionKey` (the rows themselves are not returned, and the run reads a bounded number of rows, reported as at most 100). A generated query can be invalid: then `status` is `INVALID_QUERY`, an `errors` field explains why, and the call still comes back as a normal result, not an error. Do not present a summary from an invalid query. Retry once with a clearer, more specific `prompt`; if it fails again use `search_docs` or `generate_nqe_query`, show the user the query and say it did not run.
4. **The `nqe` summary is prose, not data.** It is written by a model and can be approximate ("more than 40 devices" where the exact count was 44). For an exact number use a `get_*` tool: `get_network_complexity` gives exact counts. Always show the generated `query` so the user can review it.
5. **Report the snapshot.** Results reflect a snapshot, not the live network. Say which snapshot and that live state may differ.
6. **`get_device_vulnerabilities` lists every CVE it checked**, each with a `result`. Only `VULNERABLE` means the device is exposed; `NOT_VULNERABLE` rows are checked and clear. Count and report exposure from `VULNERABLE` rows only, by severity.
7. **Names match loosely and lookups can be empty.** Interface names match by alias (`Ethernet1` finds `et1`). A lookup that finds nothing (`get_vlan`, `get_host`) returns an empty list, not an error: say "not found in this snapshot". `get_vrf` and `get_subnet_locations` can return long lists: summarize, do not paste.
8. **An error or an empty result is a finding.** Do not fill the gap from memory, and do not paste raw error text into tickets or chat; summarize it.

## Errors
Two kinds. HTTP errors come back before any tool runs; tool errors arrive as a normal MCP result with `isError: true` and a plain-text message.

| Symptom | Meaning | Tell the user |
|---|---|---|
| HTTP 403 on every call, body `MCP is not enabled for this organization` | The organization's MCP server toggle is off, or the license is expired or inactive | MCP is disabled for this org; an administrator must re-enable it |
| HTTP 400 on `tools/list`, or 404 on a later call | The session id from `initialize` (the `Mcp-Session-Id` header) was not sent back, or is unknown | Send the id on every call after `initialize` (the live server is session-based, whatever its documentation says about being stateless) |
| HTTP 401 | The API token is wrong or revoked | Mint a new token and update the client |
| `No processed snapshot for network N` | The network has no processed snapshot, or the ID does not exist (the same message for both; the token may also lack access) | Check the ID and the token's access; do not try other IDs |
| `'networkId' is required` | The call omitted `networkId` | Ask the user for it |
| `Device 'X' not found in snapshot S`, `CVE-... not found` | The object is not in that snapshot | Say so, with the snapshot; do not guess a near match |
| `Either 'fromDevice' or 'srcIp' must be specified` | `get_paths` had neither | Ask where the traffic starts |
| `The tool failed to execute` | No detail from Forward (seen for a cloud object ID it could not find) | Say the lookup failed, check the ID, try another tool for the same object |
| A tool is missing from `tools/list` | Not licensed, or AI is not enabled for the org | Say it is unavailable here |

Clients that only support OAuth connectors cannot connect: Forward authenticates with a static Basic header only.

## Boundaries
This answers what Forward's model says. It does not list networks or snapshots; compare snapshots; list or run checks; read device configuration files; list topology links; change anything in Forward or on a device; or replace live device state. If the question needs one of those, say the MCP has no tool for it.

## Safety
Never put the API token, its secret or the base64 of `accessKey:secret` in repository files, tickets or chat, and never print the `Authorization` header: the base64 value decodes to the secret.
