# NQE gotchas: speed, silent wrong answers, missing values

Read when a query is slow, returns fewer rows than expected, or fails on a missing value. Each item says how it was checked (live = run against a real network on 2026-10-06). **Verified** means against this repository's NQE schema or builtins; **unverified** means a colleague measured it on one network, so confirm it with `validate-nqe-query` (or `fwdctl nqe run`) before stating it as fact.

## Contents

- Speed
- A query that returns too little and says nothing
- Assert on positive content
- Missing values
- Time
- Model facts that trip queries

## Speed

All **unverified**:
- A query that references `network.devices` once can run device by device in parallel, which is much faster than one that walks the network a second time (a second reference, even in a top-level definition, can turn the parallel plan off). Walk the network once and carry the device through.
- An equality test on a key in its own `where` (`foreach d in network.devices where d.name == key`) can become an indexed lookup. Keep the equality as a separate clause, not folded into a longer condition.
- Build a list once at device level only when several per-row expressions walk the same collection. Unused `let` bindings cost nothing.
- `matches` can raise on some inputs, so place its `where` by hand rather than relying on the optimizer to reorder it.
- `fwdctl` does not show the query plan; Forward's UI Debug Panel does. If a query is slow, say so and ask for the plan rather than guessing.

Reported by another session that measured them (a large FortiGate, Forward build 261012; the mechanisms are from reading Forward's source at one commit, not profiled): the effect is measured, the mechanism is **medium confidence**, and none of this was re-measured here.
- **Copy a command's response before matching it.** `regexMatches` or `substring` called directly on `command.response` (a device's collected command output) pay a cost proportional to the text before each match, because the collected string carries its exact source region. `let text = "" + command.response` and matching on `text` gave identical rows: a 400-way alternation scan took 18.0 s directly and 1.6 s on the copy, and a scan with thousands of matches took 427 s (a client timeout) against 0.57 s. `fwdctl nqe lint --experimental perf-command-response` warns about the direct form.
- **Select a device by an exact name, directly on the first loop.** `foreach device in network.devices where device.name == X` can become a keyed lookup. A filter that is not an exact equality (`toLowerCase(device.name) == toLowerCase(param)`) makes the device-parallel plan prepare the whole per-device expression for every device before the filter runs: a fixed 30.6 s when no device matched, against 12.1 s after resolving the name first and then filtering `device.name == resolved`, with byte-identical output. No lint rule: most queries legitimately filter devices by something else.
- **A large function called from many places is prepared at every call site.** Forward lowers a function body per call site with no memoization: moving eight-level tree builders from 12 call sites to 1 cut a preparation step from 5.8 s to about 1.1 s, and merging duplicated calls across the two arms of an `if` (both arms are prepared) cut 8.1 s to 5.5 s. The threshold is not known; call a large function once and pass its result.
- **Joins on a name between let-bound bags are already hash lookups** (144,500 lookups against 1,445 records cost about 0.2 s): do not restructure them. **Nothing is cached between executions**, a saved query run by id included.

## A query that returns too little and says nothing

**Unverified:** a block-pattern capture typed too narrowly (for example `{x:number}`) skips lines whose token is hex or non-numeric, so the result is short with no error. Prefer `:string` and convert afterwards. Cross-check the row count against the number of blocks (`parseConfigBlocks`, `blockMatches` exist in the builtins) before trusting a count. A query that has only ever returned zero rows is unvalidated, not correct.

## Assert on positive content

A device that does not have a feature often answers a command with a banner, an error or truncated text. A "keyword absent" check then passes on that junk. For command-output queries (`device.outputs.commands`, `commandText`, both in the schema), require a positive marker that proves the output was the right one, and add a canary column (a value that must be present) so a parse failure is visible. Say which command was read and on how many devices.

## Missing values

**Verified** on a live network (`validate-nqe-query`, 124 devices): selecting a field from a value that can be missing does not fail at run time, it fails to **compile**, and the message names the fix: "Cannot select field 'x' from null value. Consider either replacing '.' with '?.' to return null or using isPresent() to check whether the value is present before selecting a field". So:
- `d.platform.osSupport.lastSupportDate` does not compile (79 of 124 devices have `osSupport`); `d.platform.osSupport?.lastSupportDate` runs and gives null for the other 45; `where isPresent(d.platform.osSupport)` keeps only the 79.
- `?.` does **not** help a `foreach`: `foreach nb in p.bgp?.neighbors` fails with "foreach was given a null list". Guard with `where isPresent(p.bgp)` before the `foreach` (that returned the network's 692 BGP neighbours).
- `length(x.y.z)` on a missing record fails the same way as any field access; guard first.

**Unverified:** negating a possibly missing boolean (`!x`) versus `x == false`: both ran without error on a boolean that was always present, so the difference was not seen; prefer `x == false` when the value can be missing. See `fwdctl describe author-nqe-query reference/syntax-and-types.md` for the language's own handling.

## Time

**Verified** (the time guide, the schema and a live run, where `now()` is "not in scope"): there is no "now" value. Use the snapshot's own time (`snapshotInfo` holds `collectionTime`) or a date parameter, because results are cached per snapshot. A subtraction of two timestamps is a `Duration`; compare it to `days(30)`, `seconds(n)` and the like.

## Model facts that trip queries

**Verified** against `nqeschema/network.json`:
- `BgpNeighbor` has no local-address or update-source field (`localAddress` and `updateSource` fail). It holds `neighborAddress`, `peerDeviceName`, `peerVrf`, `peerRouterId`, `sessionState`, `enabled`, `description`, `peerAS`, `localAS`, `peerType` and `statistics`. The update source is in the configuration text (`reference/config-patterns.md`).
- Interface names are vendor-shortened and `aliases` casing differs by vendor: compare with `toLowerCase`.
- `osSupport` is missing on some devices (verified: 45 of 124 on one network), so guard it as above.
- Traffic-engineering tunnels (`teTunnel`) carry their label stacks (`mplsLabels`, `pushedMplsLabels`), which is where a segment-routing node SID can be read.
- A protocol the model does not cover can still show up as `originProtocol` on a next hop: enumerate across every device before saying a protocol is absent.

**Verified live** (one network, 692 BGP neighbours): `peerRouterId` is `0.0.0.0` on neighbours that are IDLE or ACTIVE (no OPEN received), and `peerVrf` is missing when `peerDeviceName` is missing (171 neighbours). `sessionState` was never missing there, so whether it can be is unverified.

**Unverified:** high-availability pair members can share addresses, so match by address, not by name.

## group-by takes one key

`group x as g by KEY as name` has exactly one key expression. `by a, b` is a syntax error (`fwdctl nqe lint` says so and hints), and a peer reported the same compile error from Forward for a tuple or record key. For several values, make one key of them (for example an interpolated string that joins them) and split it again in the `select`, or group in two steps (group by the first, then group each group by the second).

## `.` on an optional record fails at run time, `?.` returns null

`fwdctl nqe lint` calls `d.platform.osSupport.lastSupportDate` valid, but Forward fails it with "Cannot select field ... from null value" on any device without an `osSupport` record. Use `d.platform.osSupport?.lastSupportDate` (null when absent) or guard with `isPresent(d.platform.osSupport)`. Verified live on a 1,409-device network where 702 devices have no lifecycle record.

## Large networks

Filter to one vendor or device group before iterating nested lists, select scalar fields rather than whole records, and page the result. The reasons, measured effects, the device-parallel shape and how to time a query are in `reference/performance.md`. `unexpected EOF` on `POST /api/nqe` has been reported on a huge snapshot with whole records selected; narrow the query. `fwdctl context schema "<field>"` lists the fields inside a record one level down (`contains`).
