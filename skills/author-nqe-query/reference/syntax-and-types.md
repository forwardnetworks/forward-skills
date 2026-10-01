# Syntax and types

Type keywords, pattern matchers, parentheses, ordering, limits and time arithmetic. Each of these compiles silently wrong or fails the whole query when misused.

- Use NQE's real type keywords in annotations and null casts: `Bool`
  (not `Boolean`), `Integer` (not `Int`), `String`, `Date`, `Duration`,
  `List<T>`. E.g. `null : Bool`, `null : List<String>`.
- Match the matcher to the pattern type. Single-line patterns (single
  backticks `` `...` ``) go with `matches()` / `patternMatches()`, whose
  match objects expose `.line.text` and named `.data` captures. Block
  patterns (triple-backtick ```` ``` ```` literals, type `PatternBlocks`)
  go with `blockMatches()` / `hasBlockMatch()`, whose match objects
  expose `.data` and `.blocks` (NO `.line`). Never pass a single-line
  pattern to `blockMatches`, and never read `.line` off a `blockMatches`
  result.
- A `foreach ... select ...` comprehension used as a value (e.g. a `let`
  binding, a record field, an `if`/`else` branch, or an argument to
  `length`/`max`/`any`) MUST be wrapped in parentheses:
  `let xs = (foreach x in ... select ...)`. To produce an empty list as
  a fallback, use the literal `[]`, not an empty `foreach`.
- `limit N` only applies to an ordered result. Always precede `limit`
  (or any deterministic sampling) with an `order by` clause.
- `order by ... natural` uses natural sort order: numbers inside a string
  are compared by numeric value, so `device2` sorts before `device10` (the
  default sort compares digit characters one at a time and puts `device10`
  first). It is valid ONLY on `String`-typed sort keys. For `Date`,
  `Timestamp`, `Duration`, or numeric keys, use plain `asc`/`desc` WITHOUT
  `natural`. Applying `natural` to a non-`String` key (e.g. `order by
  "EOL Date" asc natural` where `"EOL Date"` is a `Date`) is a type error
  and fails the whole query. In a multi-key `order by`, attach `natural`
  only to the individual `String` keys (e.g. `order by "Days until End of
  support" asc, Device asc natural`).
- Time arithmetic is strictly typed: `Date`, `Timestamp`, and `Duration`
  are different types. Subtraction requires BOTH operands to be the same
  type: `Timestamp - Timestamp` -> `Duration` and `Date - Date` ->
  `Duration`. NEVER subtract mixed types such as `Timestamp - Date` or
  `date(ts) - someTimestamp` (use `date(...)` to coerce a `Timestamp` to
  a `Date` only when comparing two `Date`s). Build a `Duration` with
  `days(n)` / `hours(n)` / `minutes(n)` / `seconds(n)`; `Timestamp -
  Duration` -> `Timestamp` and `Date - Duration` -> `Date`.
- To test "older than N days", compare same-typed values directly
  instead of dividing — e.g.
  `where collectionTime - lastUsed > days(30)` (Duration > Duration), or
  equivalently `where lastUsed < collectionTime - days(30)` (Timestamp <
  Timestamp). `collectionTime` is `device.snapshotInfo.collectionTime`.
  If you need the elapsed time itself, subtract the two timestamps -
  `(collectionTime - lastUsed)` yields a `Duration`. A `Duration` renders
  as its non-zero components with unit suffixes, largest-first and with
  full units rolled up (e.g. `46m`, `23h 56m 4s`, `7d 3h 4m 5s`; exactly
  24h shows as `1d`), not as a single number. A `Duration` cannot be
  converted to a numeric day count: the `/` operator requires
  `Integer`/`Float` operands, so dividing a `Duration` (e.g.
  `(collectionTime - lastUsed) / days(1)`) fails with "Expression is not a
  Integer nor a Float". Compare durations directly as shown above instead.

Study the worked examples below, then produce the requested query.
