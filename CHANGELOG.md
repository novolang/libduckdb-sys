# Changelog

All notable changes to libduckdb-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-18

The documentation and comments in plain prose; no declaration changed.

### Corrected against the DuckDB C API reference

- DuckDB runs in the calling process, so a prepared statement is a
  query the library has parsed and planned rather than a server.
- A parameter place is written `?` for a positional parameter or
  `$name` for a named one, and the positional places are numbered
  from 1.
- The result of an `INSERT` has one column, named `Count`, whose
  single value is the number of rows the statement changed.
- The suite holds ten tests, not nine.

## 0.1.0 — 2026-09-16

The first release: forty-four entry points of the DuckDB C API, one
`@ffi` declaration each, and no logic.

### Added

- `libduckdb` — the whole surface, in six groups.
  - The database and the connection: `duckdb_open`, `duckdb_open_ext`,
    `duckdb_close`, `duckdb_connect`, `duckdb_disconnect`,
    `duckdb_interrupt` and the library version.
  - The configuration: `duckdb_create_config`, the setting count, the
    setting names, `duckdb_set_config` and `duckdb_destroy_config`.
  - A query and its result: `duckdb_query`, `duckdb_destroy_result`,
    the column count and names and types, the row count, the rows
    changed, and the error message and its kind.
  - Reading a value: `duckdb_value_is_null` and the boolean, 32-bit,
    64-bit, unsigned, double and text readers.
  - The prepared statement: `duckdb_prepare`, its error, the parameter
    count, names and types, the six binds, `duckdb_clear_bindings` and
    `duckdb_execute_prepared`.
  - Memory: `duckdb_malloc` and `duckdb_free`.
- `tests/libduckdb_tests.nv` — ten tests over the entry points. Every
  database they open is `":memory:"`, so the suite reads and writes no
  file and needs no privileges.

### The handle discipline, which is this package's decision

`duckdb_database`, `duckdb_connection`, `duckdb_prepared_statement` and
`duckdb_config` are each a pointer to an opaque structure. That part is
ordinary. A caller carries an address and never looks inside.

What is not ordinary is where the address lives. A handle is produced
through an out-parameter and destroyed through the address of the slot
that holds it. `duckdb_open` takes the address of an eight-byte slot and
writes the handle into it, and `duckdb_close` takes that same address,
releases the database and writes zero back. Every call in between takes
the handle itself, read out with `ptr.read_word`. A novo-lang caller
therefore keeps the slot rather than the handle for as long as the thing
lives.

`duckdb_result` is different again, and it is the only structure the
caller owns outright. It is not a pointer to an opaque thing. It is a
structure the caller reserves the bytes for, hands to `duckdb_query` by
address, reads only through the accessors, and releases with
`duckdb_destroy_result`. The current header lays it out as six
eight-byte words. The C API promises no size for it and offers no call
that reports one, so the README tells a caller to reserve more than the
six words and never to read a field out of it by offset.

### The effect rows

A call that opens a store, runs a query or releases what a query holds
is `[io, ffi]`. Reading a value out of a result that is already in
memory, binding a value into a statement, and asking the library its
version or its settings are `[ffi]`.

### Unverified

DuckDB was not installed on the machine this package was written on.
`novo pkg build` type-checks the declarations without it and is green.
`novo test` stopped at `cannot find -lduckdb`, so the suite has never
been linked and no assertion in it has ever been observed to hold. The
declarations were checked against the DuckDB C API reference. Treat the
whole package as unmeasured until someone runs it against an installed
libduckdb.

### Named as missing

**The data chunk interface, and it is the largest omission in this
package.** DuckDB's current way of reading a result is
`duckdb_fetch_chunk`, which hands back a chunk of up to 2048 rows in
columnar form. It takes a `duckdb_result` by value, and the novo-lang
foreign function interface passes integers, floats and strings, so it
cannot be declared. Nothing a chunk leads to can be
reached without it: `duckdb_data_chunk_get_vector`,
`duckdb_vector_get_data`, `duckdb_vector_get_validity` and
`duckdb_validity_row_is_valid` are all absent although each of them
would bind perfectly well on its own. `duckdb_result_chunk_count`,
`duckdb_result_get_chunk`, `duckdb_result_is_streaming` and
`duckdb_result_return_type` take a `duckdb_result` by value for the same
reason.

That is why the value readers in this package are the ones the C header
marks deprecated. `duckdb_value_int64` and its neighbours take the
result by address, and they are the only readers that do. The package
binds them knowing they are deprecated, because the alternative is a
package that can run a query and not read its answer.

**`duckdb_string_is_inlined` and the `duckdb_string_t` accessors.** They
take a `duckdb_string_t` by value. A caller reaches them only through a
vector, which is already absent.

**The small value structures.** `duckdb_hugeint`, `duckdb_uhugeint`,
`duckdb_decimal`, `duckdb_interval`, `duckdb_date_struct`,
`duckdb_timestamp_struct` and `duckdb_blob` are passed and returned by
value, so `duckdb_value_hugeint`, `duckdb_value_date`,
`duckdb_value_timestamp`, `duckdb_value_blob`, `duckdb_from_date`,
`duckdb_to_date` and `duckdb_hugeint_to_double` have no declarations. A
column of one of those types is read as text with
`duckdb_value_varchar`.

**`duckdb_value_float`.** It answers a C `float`, and the novo-lang
`Float` is a C `double`. Rather than declare a return of the wrong
width, the package leaves it out: `duckdb_value_double` reads a `REAL`
column and converts it.

**Everything built on a C function pointer.** The replacement scans, the
user-defined table, scalar and aggregate functions, and the destructor
callbacks registered with them.

**The Arrow interface.** `duckdb_query_arrow` and its neighbours pass
`ArrowSchema` and `ArrowArray` structures, and the C header marks them
deprecated.

**The appender, the pending and streaming results, and the
`duckdb_value` object interface.** Left out of the first release.
