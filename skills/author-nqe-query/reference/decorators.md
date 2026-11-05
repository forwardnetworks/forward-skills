# NQE decorators

A decorator attaches the rows of a saved NQE query to Forward's device, host, interface or path views (NQE > Decorators in the UI, or `edit-platform` area `decorators`: create, update, enable, disable; `inspect-platform` area `decorators` lists them and checks each one's columns against its query). Summarised from Forward's documentation (Decorators, under NQE); check it on the Forward you target.

- **Device, host and interface decorators**: an ordinary query that takes **no parameters** and runs **once**; Forward joins its rows to each entity by the **join columns** chosen when the decorator is created. A join column of the wrong NQE type makes the decorator invalid.
  - Device: one `String` column of device names.
  - Interface: two `String` columns, device name and interface name.
  - Host: an `IpAddress` column (required; an address can match several hosts), and optionally a `String` host name, a `String` device name and a `MacAddress` column, each of which narrows the match. A host's `addresses` are `IpSubnet`, so use `address(subnet)` for the IP column.
- **Path and path-interface decorators**: different: the query runs **once per path**, takes one parameter of type `Path`, and is marked `@query`.

Check a decorator query like any other: `fwdctl nqe lint` (the types of the join columns are in the result row type), then `validate-nqe-query` to see real rows, then save it with `edit-nqe-query` and create the decorator with `edit-platform` area `decorators` (dry run first; it checks the columns against the saved query before anything is sent, since Forward saves a bad decorator without complaint and reports it only when it decorates).

## Cost, scope and when it runs (read from Forward's source, not measured)

- A device, host or interface decorator's query runs **once for the whole snapshot**, not per device, per host or per page. Forward indexes the result and reuses it: each page only looks up its own entities. The index is rebuilt for a new snapshot, a new commit of the query, or a change to the decorator's columns. Forward gives the query 5 minutes.
- The whole result is held as one index, so its size is the cost: before creating a decorator, count its rows (`fwdctl nqe run --query-id <id> --network <n> --limit 1 --meta -` and read `total`) and narrow the query to the rows and columns that matter. A query of hundreds of thousands of rows is a real load on every new snapshot.
- **Enabled** decides whether Forward's views use it: the UI asks only for enabled decorators of the entity type, so a disabled decorator is stored but not computed by page loads. An API decorate call that names a user decorator runs it even when disabled.
- `metadataColNames` empty (or left out) shows every column of the rows.
- To remove a decorator: disable it (`edit-platform` area `decorators` action `disable`), or delete it in the Forward UI (NQE > Decorators).

