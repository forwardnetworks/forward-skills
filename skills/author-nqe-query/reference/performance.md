# Writing NQE that is fast and scales

A valid query can still take hours on a large network. Check these before you run one over many devices; `fwdctl nqe lint` reports the structural ones as `info`. Sources: Forward's NQE performance guide (the rules) and measurements by another session (marked *measured*); direction carries over, exact numbers do not.

## 1. Make it device-parallel (the biggest win)

NQE runs one task per device, in parallel, when the **main comprehension starts `foreach d in network.devices`** and:
- `network` is only used to select fields (never passed to a function);
- `network.devices` is named **once**;
- the main comprehension has **no `group`** (group inside a per-device `let`, or aggregate in a second query).

That is typically 10 to 100 times faster **for a query over many devices**. For a request that selects one device (or a few) and does heavy work per device, look the device up by exact name instead: `foreach d in names foreach device in network.devices where device.name == d` is a keyed lookup and was measured faster (30.6 s down to 12.1 s on a large network) than the device-parallel path with a computed filter. The Query Debug Panel in the NQE Library (option-click Execute) shows the optimized plan: a device-parallel query starts `parallel_foreach`.

## 2. Help the optimizer

- **Filter early**: put each `where` right after the `foreach` it depends on. NQE floats `where` and `let` upward only when the condition cannot raise an error (division, list index), so keep conditions total.
- **Equality joins are already lookups**: `foreach b in list where b.key == a.key` becomes a dictionary lookup when the list is from the data model or reused. Do not hand-build an index.
- **Filter devices by exact name**: `device.name == X` is a keyed lookup; a computed filter (`toLowerCase(device.name) == ...`) evaluates the per-device work for every device first. Resolve names in a small first query when you must normalise them. *Measured:* about 2.5x on a large network.
- Unused `let`s are removed for you.

## 3. Cost follows reachable code, not executed code

*Measured:* every branch and call site reachable from the entry is prepared, executed or not. A large query did nothing for 0.16 s when its entry did nothing, and spent most of its time preparing branches that never ran.
- Give a large function **one call site**: build everything in one comprehension and look results up by key.
- Do not repeat the same call in both arms of an `if`; compute it once.
- Prefer the **native model** (for example `securityPolicy`) to parsing configuration text. *Measured:* roughly 2x.

## 4. Text and regex

*Measured:* `regexMatches` or `substring` called directly on a collected command's response costs per match, not per scan, and was 10 to 700 times slower than on a copy. **Copy the string first**: `let text = "" + command.response`. `fwdctl nqe lint --experimental perf-command-response` flags it.

## 5. Keep results and queries small

- **Select scalar fields, not whole records**, and filter to one vendor or device group before iterating nested lists on a very large network.
- A record field over 1,000,000 bytes fails the whole request; bound per-item size and page (a few hundred rows per batch).
- A very large **inline** query is analysed again on every run. Save it to the NQE library and run it **by id** (`fwdctl nqe run --query-id`).
- A `TIMED_OUT` or `SYSTEM_ERROR` result is cached: an identical retry fails immediately (`--meta` says "cached failure"); change or narrow the query first, or add `--fresh` to make the program different without changing the rows.
- `unexpected EOF` on `POST /api/nqe` was reported once on a huge snapshot with whole records selected (cause not established): narrow the query.
- **A job that needs every row of a large network** should not first learn the result is too big: ask for one page (`--limit 1 --meta m.json`; the page line and `estimated_total_bytes` give the total rows and the size as text), and over 100 MB pass `--allow-large` with output to a temp file, then hash or aggregate as it streams, or page with `--limit/--offset`. Without `--allow-large` the guard refuses only after the query has run, and the work is lost.
- Parallel requests share a fixed set of workers; four at once gave about 2x, not 4x.

## 6. Measure

- `fwdctl nqe explain --query-id Q_... ` prints Forward's own optimized plan for a saved query (the Query Debug Panel's text: `parallel_foreach`, `_lookup`, totality and cardinality marks). `fwdctl nqe metrics` lists the slow runs Forward recorded (needs its DEBUG_NQE permission).
- `fwdctl nqe run --meta FILE` (runs are asynchronous by default) records Forward's execution time (`millis_executing`: not parse, optimize or queue time), the outcome and whether the result was cached. A synchronous run only gives your own wall-clock time.
- A result is cached on (optimized program, snapshot, parameters): for a cold timing change the query (an always-true `where "<unique>" != ""` works; comments do not), and compare versions one at a time.
- Check two versions return identical rows before keeping the faster one.
