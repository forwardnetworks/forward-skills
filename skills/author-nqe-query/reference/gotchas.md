# NQE gotchas: speed, silent wrong answers, missing values

Read when a query is slow, returns fewer rows than expected, or fails on a missing value. Each item says how it was checked. **Verified** means against this repository's NQE schema or builtins; **unverified** means a colleague measured it on one network, so confirm it with `validate-nqe-query` (or `fwdctl nqe run`) before stating it as fact.

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

## A query that returns too little and says nothing

**Unverified:** a block-pattern capture typed too narrowly (for example `{x:number}`) skips lines whose token is hex or non-numeric, so the result is short with no error. Prefer `:string` and convert afterwards. Cross-check the row count against the number of blocks (`parseConfigBlocks`, `blockMatches` exist in the builtins) before trusting a count. A query that has only ever returned zero rows is unvalidated, not correct.

## Assert on positive content

A device that does not have a feature often answers a command with a banner, an error or truncated text. A "keyword absent" check then passes on that junk. For command-output queries (`device.outputs.commands`, `commandText`, both in the schema), require a positive marker that proves the output was the right one, and add a canary column (a value that must be present) so a parse failure is visible. Say which command was read and on how many devices.

## Missing values

**Unverified** (check each error message on a live run): operations on a missing value stop the whole query instead of yielding nothing, including a `foreach` over it, `length(x)`, a duration built from it and a field access on it. Guard with `isPresent` (a builtin) in a `where` before the use; compare with `x == false` rather than negating a possibly missing boolean. See `fwdctl describe author-nqe-query reference/syntax-and-types.md` for the language's own handling.

## Time

**Verified** (the time guide and schema): there is no "now" value. Use the snapshot's own time (`snapshotInfo` holds `collectionTime`) or a date parameter, because results are cached per snapshot. A subtraction of two timestamps is a `Duration`; compare it to `days(30)`, `seconds(n)` and the like.

## Model facts that trip queries

**Verified** against `nqeschema/network.json`:
- `BgpNeighbor` has no local-address or update-source field (`localAddress` and `updateSource` fail). It holds `neighborAddress`, `peerDeviceName`, `peerVrf`, `peerRouterId`, `sessionState`, `enabled`, `description`, `peerAS`, `localAS`, `peerType` and `statistics`. The update source is in the configuration text (`reference/config-patterns.md`).
- Interface names are vendor-shortened and `aliases` casing differs by vendor: compare with `toLowerCase`.
- `osSupport` can be missing for lab images, so guard it.
- Traffic-engineering tunnels (`teTunnel`) carry their label stacks (`mplsLabels`, `pushedMplsLabels`), which is where a segment-routing node SID can be read.
- A protocol the model does not cover can still show up as `originProtocol` on a next hop: enumerate across every device before saying a protocol is absent.

**Unverified:** high-availability pair members can share addresses, so match by address, not by name; `peerRouterId` can be `0.0.0.0` when no OPEN was received; `peerVrf` is missing when `peerDeviceName` does not resolve.
