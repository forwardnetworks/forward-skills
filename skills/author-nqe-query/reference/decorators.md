# NQE decorators

A decorator attaches the rows of a saved NQE query to Forward's device, host, interface or path views (NQE > Decorators in the UI, or `edit-platform` area `decorators`: create, update, enable, disable; `inspect-platform` area `decorators` lists them and checks each one's columns against its query). Summarised from Forward's documentation (Decorators, under NQE); check it on the Forward you target.

- **Device, host and interface decorators**: an ordinary query that takes **no parameters** and runs **once**; Forward joins its rows to each entity by the **join columns** chosen when the decorator is created. A join column of the wrong NQE type makes the decorator invalid.
  - Device: one `String` column of device names.
  - Interface: two `String` columns, device name and interface name.
  - Host: an `IpAddress` column (required; an address can match several hosts), and optionally a `String` host name, a `String` device name and a `MacAddress` column, each of which narrows the match. A host's `addresses` are `IpSubnet`, so use `address(subnet)` for the IP column.
- **Path and path-interface decorators**: different: the query runs **once per path**, takes one parameter of type `Path`, and is marked `@query`.

Check a decorator query like any other: `fwdctl nqe lint` (the types of the join columns are in the result row type), then `validate-nqe-query` to see real rows, then save it with `edit-nqe-query` and create the decorator with `edit-platform` area `decorators` (dry run first; it checks the columns against the saved query before anything is sent, since Forward saves a bad decorator without complaint and reports it only when it decorates).
