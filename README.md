# libduckdb-sys

DuckDB is an in-process analytical database. It runs inside the program
that uses it, like SQLite, and it answers analytical queries — scans,
aggregations and joins over columns — the way a data warehouse does.
It speaks SQL, it reads and writes its own file format, and it can query
CSV and Parquet files where they lie. The C interface is documented in
the [DuckDB C API reference](https://duckdb.org/docs/stable/clients/c/overview).
This package declares forty-four of that interface's entry points to
novo-lang, one declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in libduckdb. The package contains no logic of
its own, and it does nothing without the C library installed. The
forty-four entry points are enough to open a database, connect to it,
run a query or a prepared statement, and read the answer value by value;
the section "What is not included" says what a program still cannot do
with them alone, and why one of the omissions is larger than it looks.

## What it is

A **database** is the store. It is a file, or the word `":memory:"` for
a store that exists only while the program runs.

A **connection** is one session against a database. It has its own
transaction state, and a database may hold several.

A **result** is the whole answer to a query, computed and held in memory
before the call returns. A result is not a cursor: nothing is left to
fetch, and the rows do not arrive as the query runs.

A **prepared statement** is a query the server has parsed and planned,
with places left for values. The places are written `?` and are numbered
from 1. Binding a value into a place is not the same as writing it into
the query text: a bound value is never parsed as SQL.

A **handle** is the address of something the library owns: a database, a
connection, a prepared statement, a configuration. The C header declares
the structure behind each one and never defines it, so a program holds
the address and cannot look inside.

## Install

```
novo pkg add libduckdb-sys
```

Adding the package does not install the C library. DuckDB publishes the
C interface as a zip holding `duckdb.h` and `libduckdb.so`, on the
[installation page](https://duckdb.org/docs/installation/). The zip is
unpacked wherever the reader wants it, and the linker is told where
that is:

```
export LIBRARY_PATH=/path/to/duckdb
export LD_LIBRARY_PATH=/path/to/duckdb
```

On macOS the Homebrew formula is `duckdb`. DuckDB installs no
pkg-config file, so there is no `libduckdb.pc` to ask and the link flag
`-lduckdb` is written into `novo.toml` instead.

## Example

A query run against a database in memory:

```novo ignore
use libduckdb

fn main() [io, ffi]
    // A handle arrives in a slot the caller owns.
    let db_slot = ptr.alloc_word()
    if libduckdb.duckdb_open(":memory:", db_slot) == 1
        println("the database did not open")
        return
    let conn_slot = ptr.alloc_word()
    let _ = libduckdb.duckdb_connect(ptr.read_word(db_slot), conn_slot)

    // The result is the one structure the caller lays out.
    let result = ptr.alloc(128)
    if libduckdb.duckdb_query(ptr.read_word(conn_slot),
                              "SELECT 42 AS n, 'duck' AS word", result) == 1
        println(ptr.read_str(libduckdb.duckdb_result_error(result)))
        libduckdb.duckdb_destroy_result(result)
        return

    println("${libduckdb.duckdb_row_count(result)} row(s)")
    println("${libduckdb.duckdb_value_int64(result, 0, 0)}")

    // The text is DuckDB's to allocate and the caller's to release.
    let word = libduckdb.duckdb_value_varchar(result, 1, 0)
    println(ptr.read_str(word))
    libduckdb.duckdb_free(word)

    libduckdb.duckdb_destroy_result(result)
    // A destroyer takes the slot, not the handle in it.
    libduckdb.duckdb_disconnect(conn_slot)
    libduckdb.duckdb_close(db_slot)
    ptr.free(result)
    ptr.free(conn_slot)
    ptr.free(db_slot)
```

The example is fenced as an illustration rather than a compiled block
because `novo doc` compiles the blocks in documentation comments and not
the ones in this file. The same calls are in
`tests/libduckdb_tests.nv`.

## What the package contains

| Module | Contents |
| --- | --- |
| `libduckdb` | Every entry point, in six groups: the database and the connection, the configuration, a query and its result, the row-at-a-time value readers, the prepared statement with its binds, and the library's own allocator. |

The six groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Database and connection | 7 | Opens a store, connects to it, interrupts a query, and closes both. |
| Configuration | 5 | Lists the settings the library has, sets them, and opens a database with them. |
| Query and result | 9 | Runs SQL, reports the columns and their types, and reports a failure. |
| Reading a value | 7 | Reads one column of one row as a boolean, an integer, a double or text. |
| Prepared statements | 14 | Parses a query once, binds values into it, and runs it. |
| Memory | 2 | The allocator every string this library hands back belongs to. |

## How to choose an entry point

`duckdb_open` is for a database with the default settings.
`duckdb_open_ext` is the same call with a configuration and an error
message; use it when a setting matters, or when knowing why an open
failed matters.

`duckdb_query` is for a query with no values to supply. It parses,
plans and runs in one call.

`duckdb_prepare` with the binds and `duckdb_execute_prepared` is for a
query that carries values, and for a query run many times. A value bound
into a place is never parsed as SQL, so it is also the way to run a
query built around data that came from outside.

`duckdb_value_varchar` reads any column as text, whatever its type.
The typed readers avoid the allocation and the conversion, and they are
the ones to use when the column type is known.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** Every handle is the
   address the library returned.
2. **A handle is produced through a slot and destroyed through that
   slot.** `duckdb_open` takes the address of an eight-byte slot from
   `ptr.alloc_word` and writes the handle into it. `duckdb_close` takes
   the address of the same slot, not the handle in it, and writes zero
   back. The same holds for `duckdb_disconnect`,
   `duckdb_destroy_prepare` and `duckdb_destroy_config`. Every other
   call takes the handle itself, read out with `ptr.read_word`.

   | Producer | Writes into | Destroyer takes |
   | --- | --- | --- |
   | `duckdb_open`, `duckdb_open_ext` | a word slot | that slot |
   | `duckdb_connect` | a word slot | that slot |
   | `duckdb_prepare` | a word slot | that slot |
   | `duckdb_create_config` | a word slot | that slot |
   | `duckdb_query`, `duckdb_execute_prepared` | a `duckdb_result` | that structure |

3. **A `duckdb_result` is a structure the caller owns.** It is not a
   handle. `duckdb_query` is handed the address of bytes the caller
   reserved, and it writes the structure into them. The current header
   lays it out as six eight-byte words, which is 48 bytes, and **the C
   interface promises no size and offers no call that reports one**, so
   reserve more than that — 128 bytes is what the test suite uses — and
   never read a field out of it by offset. Every field is read through
   an accessor: `duckdb_column_count`, `duckdb_row_count`,
   `duckdb_result_error` and the value readers.
4. **A failed query fills the result too.** `duckdb_query` answers 1
   and still writes a result, because that is where the message is.
   Call `duckdb_destroy_result` whether the query succeeded or not.
5. **A `duckdb_state` is 0 for success and 1 for failure.** There is no
   third value, and there is no error code: the message is text.
6. **A C `bool` is tested against 0.** `duckdb_value_is_null` and
   `duckdb_value_boolean` answer one byte widened to an `Int`. Write
   `!= 0`, not `== 1`.
7. **An entry point that answers a 32-bit integer answers it in 32
   bits.** Write `as i32` before comparing `duckdb_value_int32` with a
   negative number.
8. **`duckdb_value_uint64` hands over the bits.** A value above the
   largest signed 64-bit number arrives negative.
9. **A NULL reads as a zero.** Every typed reader answers 0, 0.0 or
   false for a NULL, and `duckdb_value_varchar` answers the null
   address. `duckdb_value_is_null` is the only way to tell a NULL from a
   zero that is really there.
10. **A string DuckDB allocated is released with `duckdb_free`.**
    That covers `duckdb_value_varchar`, `duckdb_parameter_name` and the
    error from `duckdb_open_ext`. `ptr.free` is a different allocator
    and must not be used on them. A string an accessor merely points at
    — `duckdb_column_name`, `duckdb_result_error`,
    `duckdb_prepare_error`, `duckdb_library_version` — belongs to the
    result or to the library and is not released at all; it is valid
    until the thing that owns it is destroyed.
11. **Parameters are numbered from 1.** `duckdb_nparams` answers how
    many there are, and a bind above that number answers 1.
12. **A failed preparation still writes a handle.** That is where
    `duckdb_prepare_error` reads the message from, and the handle is
    destroyed the same way a successful one is.
13. **The column type numbers are numbers**, because the C header
    spells them as an enumeration.

    | Type | Number |
    | --- | --- |
    | `DUCKDB_TYPE_INVALID` | 0 |
    | `DUCKDB_TYPE_BOOLEAN` | 1 |
    | `DUCKDB_TYPE_TINYINT` | 2 |
    | `DUCKDB_TYPE_SMALLINT` | 3 |
    | `DUCKDB_TYPE_INTEGER` | 4 |
    | `DUCKDB_TYPE_BIGINT` | 5 |
    | `DUCKDB_TYPE_UBIGINT` | 9 |
    | `DUCKDB_TYPE_FLOAT` | 10 |
    | `DUCKDB_TYPE_DOUBLE` | 11 |
    | `DUCKDB_TYPE_TIMESTAMP` | 12 |
    | `DUCKDB_TYPE_DATE` | 13 |
    | `DUCKDB_TYPE_HUGEINT` | 16 |
    | `DUCKDB_TYPE_VARCHAR` | 17 |
    | `DUCKDB_TYPE_BLOB` | 18 |

14. **A result is the whole answer.** It is computed and held in memory
    before `duckdb_query` returns, so a query over a large table costs
    that memory. There is no call here that streams one.
15. **Every connection is disconnected before the database is closed.**

## What is not included

- **The data chunk interface, and this is the large one.** DuckDB's
  current way of reading a result is `duckdb_fetch_chunk`, which hands
  back a chunk of up to 2048 rows in columnar form, and it takes a
  `duckdb_result` **by value**. The novo-lang foreign function interface
  passes integers, floats and strings, so there is no way to call it,
  and with it go `duckdb_data_chunk_get_vector`,
  `duckdb_vector_get_data`, `duckdb_vector_get_validity` and everything
  else a chunk leads to. `duckdb_result_chunk_count`,
  `duckdb_result_get_chunk`, `duckdb_result_is_streaming` and
  `duckdb_result_return_type` take a `duckdb_result` by value for the
  same reason. That is why the row-at-a-time readers in this package are
  the ones the C header marks deprecated: they take the result **by
  address**, and they are the only readers that do.
- **`duckdb_string_is_inlined` and the `duckdb_string_t` accessors.**
  They take a `duckdb_string_t` by value. A caller reaches them only
  through a vector, which is already absent.
- **Everything that passes a small value structure.** `duckdb_hugeint`,
  `duckdb_uhugeint`, `duckdb_decimal`, `duckdb_interval`,
  `duckdb_date_struct`, `duckdb_timestamp_struct` and `duckdb_blob` are
  passed and returned by value, so `duckdb_value_hugeint`,
  `duckdb_value_date`, `duckdb_value_timestamp`, `duckdb_value_blob`,
  `duckdb_from_date`, `duckdb_to_date`, `duckdb_hugeint_to_double` and
  their neighbours have no declarations. A column of one of those types
  is read as text with `duckdb_value_varchar`.
- **`duckdb_value_float`.** It answers a C `float`, and the novo-lang
  `Float` is a C `double`. `duckdb_value_double` reads a `REAL` column
  and converts it.
- **The appender**, `duckdb_appender_*`. It is the fast path for
  inserting many rows, and it is left out of the first release.
- **The extraction of multiple statements**,
  `duckdb_extract_statements`, and the pending and streaming result
  interfaces.
- **Everything built on a C function pointer.** The replacement scans,
  the user-defined table functions, the user-defined scalar and
  aggregate functions, and the destructor callbacks a caller registers
  with them. The foreign function interface takes no function pointer.
- **The Arrow interface**, `duckdb_query_arrow` and its neighbours. It
  passes `ArrowSchema` and `ArrowArray` structures, and it is deprecated
  in the C header.
- **The `duckdb_value` object interface**, `duckdb_create_int64`,
  `duckdb_get_varchar` and the rest, and the logical type constructors
  they go with.

## Related packages

There is no novo-lang port of DuckDB and there will not be one: an
analytics engine is a query planner, an execution engine and a storage
layer, not a file format someone can implement in an afternoon.

`sqlite-nv` and `libsqlite3-sys` are the row-store neighbours, for a
program that reads and writes records one at a time. `parquet-nv` and
`arrow-nv` are the columnar file formats, for a program that needs to
read the files DuckDB reads without running a database at all.

## Tests

`tests/libduckdb_tests.nv` holds nine tests written against the
signatures. They call the C library, so `novo test` needs libduckdb
installed and linkable:

```
novo test tests/libduckdb_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

**Unverified: the suite has never linked on the staging machine.**
DuckDB was not installed where this package was written, so `novo test`
stopped at `cannot find -lduckdb` and no assertion below has ever been
observed to hold. Every assertion is written from the documented C API.
Treat the package as unmeasured until someone runs it against a real
libduckdb.

Every database the suite opens is `":memory:"`, so it reads and writes
no file and needs no privileges. The handle test asserts that a
destroyer writes zero back into the slot it was given. The configuration
test asserts that a setting which does not exist is refused rather than
ignored. The query test reads one row as an integer, a double, a boolean,
a NULL and text, and asserts that an integer reads as text too. The
unsigned test asserts that the largest 64-bit value arrives as -1. The
failure test asserts that a failed query still fills the result with its
message. The prepared-statement test binds all six kinds, runs the
statement, and asserts that a bind above the parameter count is refused.

## Implementation status

| Group | State |
| --- | --- |
| Database and connection | Complete. |
| Configuration | Complete. |
| Query and result | Complete for a materialised result. |
| Reading a value | Complete for boolean, 32- and 64-bit integers, double and text. |
| Prepared statements | Complete for the six binds above. |
| Memory | Complete. |
| The data chunk interface | Absent. Its entry point takes a result by value. |
| Dates, timestamps, decimals, hugeints and blobs | Absent. They are passed by value. Read them as text. |
| The appender | Absent. Left out of the first release. |
| Streaming and pending results | Absent. |
| User-defined functions and replacement scans | Absent. They take C function pointers. |
| The Arrow interface | Absent. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

DuckDB itself is distributed under the MIT licence, and installing it is
the reader's own step.
