# sqlite-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

The SQLite file format, read natively by novo-lang, with no
libsqlite3 anywhere in the build.

A SQLite database is the most widely exchanged binary format there is
— a phone backup, a browser profile, a mail store, a firmware dump, an
attachment somebody sent you.  Reading one should not require a C
toolchain, and on the grid it does not: this package is the format
itself, from the hundred bytes at the front of page 1 down to the
overflow chain at the end of a BLOB.

The `libsqlite3-sys` row on [the bindings shelf](https://novo-lang.org/docs/orbit-map.html)
is the escape hatch this package exists to make unnecessary for
READING.  That row stays on the shelf, because a C library that is the
reference implementation for something as large as SQLite's query
engine is worth having beside the port; but a program that wants to
read a database file should not have to take it, and after this
package lands it will not.

Nine modules, and a reader should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **header and the offsets** | `sqfmt` | anything. Start here |
| the **row** | `sqrec` | you are decoding a record |
| the **tree** | `sqbtree` | you are walking a table or an index |
| the **log** | `sqwal` | the database is in WAL mode |
| the **catalogue** | `sqschema` | you need to know what the tables are |
| the **file** | `sqfile` | you want the pages actually read |
| the **queries** | `sqdriver` | you want SQL and a `dyn Database` |
| the **commit** | `sqwrite` | you want to change something |
| the **faults** | `sqerror` | something would not decode |

## Adding it, and checking it

```bash
novo pkg add sqlite-nv           # into your novo.toml
novo pkg build                   # type- and effect-check the package
novo test --isolate tests/sqfmt_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: sqlite-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use sqdriver
use sqrec

// Every row of one table, out of a file somebody else wrote.
fn dump(path: Str, table: Str) -> [Str] [fs]
    match sqdriver.open_read_only(path)
        Err(f) => []
        Ok(db) =>
            match sqdriver.table_rows(db, table)
                Err(f)   => []
                Ok(rows) => list.map(rows, render)

fn render(row: [sqrec.SqValue]) -> Str
    str.join(list.map(row, sqrec.show), " | ")
```

Nothing in that mentions a page.

## The layer, and why

`host` — and exactly two modules earn it.

`sqfile` opens the file, reads a page at an offset, takes the four
lock bytes and fsyncs; `sqdriver` and `sqwrite` sit on top of it.
Everything else — `sqfmt`, `sqrec`, `sqbtree`, `sqwal`, `sqschema`,
`sqerror` — is `[]` throughout, and is written so that moving it to a
`core` sibling would cost a manifest edit and nothing else.

That is not a promise to move it.  The format is ONE format, and
splitting the reader from the file it reads would leave a consumer
assembling two packages to open a database.  It is a statement of
where the boundary is, so a reader can see that the arithmetic over an
untrusted file is testable with no file in the test — and
`tests/sqfmt_tests.nv`, `tests/sqrec_tests.nv`, `tests/sqbtree_tests.nv`,
`tests/sqwal_tests.nv` and `tests/sqschema_tests.nv` are that claim
exercised: thirty-seven assertions and not one file handle.

The other direction is the one worth stating: a `core` package may not
depend on this one, so nothing on the grid can acquire a file handle
by depending on a SQLite reader.

**No device claim.**  A page is 4096 bytes and a b-tree descent holds
several of them; a microcontroller that wants a key-value store on
flash wants bbqueue-nv or lsm-nv's `core` half, not a SQLite reader.

## The load-bearing interface

`sqbtree.SqPageRequest` — **the only way anything in this package
obtains a byte of a database is by naming a page number and being
handed that page.**

```novo norun:pseudo
pub enum SqPageRequest
    SqNeedPage(page_no: Int)
    SqPutPage(page_no: Int, bytes: [Int])
    SqTakePage
    SqDropPage(page_no: Int)
```

Nothing in `sqbtree`, `sqfmt`, `sqrec`, `sqwal` or `sqschema` opens,
seeks, maps or reads.  `sqfile` does all four and answers.

**Why that is the shape for this format in particular.**  The
realistic input to this package is a file somebody else wrote, and
possibly wrote on purpose.  A reader that held a byte buffer and an
offset has exactly one defence against a hostile file — checking every
offset it computes — and it has to get every single one of them right.
A reader that can only say *give me page 47* has a different property:
the worst a corrupt interior page can do is name a page that does not
exist, and `sqfile` answers `SqNoSuchPage` because it knows the file's
length and the cursor does not.  Out of range is answered by the party
that can answer it, once, instead of being re-checked at every decode
site.

It also means a consumer can put the file anywhere.  A database inside
a tar archive, in object storage, in a memory image, behind a network
block device: `sqbtree.step` and `sqbtree.feed_page` are public, and
`sqfile.pump` is one convenience over them rather than the only door.

**Two rules make the requests correct rather than merely bounded**, and
both are public functions in `sqfmt` for the same reason the request
is public — they are where a reader goes wrong quietly:

- **`usable_size` is `page_size - reserved`, not `page_size`.**  Byte
  20 of the header is zero in almost every file anybody has, which is
  exactly why a reader that used the page size passes every test it is
  given and then reads a payload short by a handful of bytes on the
  first encrypted or checksummed database it meets.  Every offset check
  in this package takes the usable size as an argument rather than
  deriving it again.
- **`local_payload_bytes` keeps the last overflow page full.**  Its
  modulus is the part that looks arbitrary and is not: a reader that
  dropped it takes the right number of bytes off the page and then
  starts the chain at the wrong offset, which produces a record that
  decodes — into the wrong values.

## What a reader gets wrong, and where each one has a name

Six places, and each is a public function rather than a comment,
because every one of them produces a WRONG ANSWER rather than an
error:

| the mistake | what it costs | where it is named |
| --- | --- | --- |
| the page size instead of the usable size | a payload short by 1–255 bytes | `sqfmt.usable_size` |
| the local-payload modulus dropped | a record that decodes into the wrong values | `sqfmt.local_payload_bytes` |
| page 1's b-tree header at offset 0 | the schema table unreadable | `sqfmt.page_header_offset` |
| serial codes 8 and 9 given a width | every column after them shifted | `sqrec.serial_bytes` |
| the `INTEGER PRIMARY KEY` column read from the record | NULL for every primary key | `sqschema.rowid_alias` |
| a `WITHOUT ROWID` table walked as a table b-tree | rowids invented out of payload lengths | `sqschema.is_without_rowid` |

And two that end a loop rather than a value: `sqbtree.depth_limit`,
which is what stops a corrupt child pointer descending forever, and
`sqwal.ends_log`, which separates a log's ENDING from a log's damage —
a recovery that calls a torn tail corruption throws away every
committed transaction in it.

## The `std.sql` driver

`sqdriver.SqDatabase` implements the prelude's `Database` trait — the
standard library's one vendor-connect contract for engines — so a
program written against `dyn Database` runs over a SQLite file this
package opened, over the standard library's own `SqliteDb`, and over
postgres-nv's client, with nothing in it naming any of the three:

```novo norun:pseudo
fn migrate(db: dyn Database) -> Str [io, fs]
    match db.exec("CREATE TABLE t (n INTEGER)")
        Some(e) => "migration failed"
        None    => "ok"
```

The contract is narrow on purpose: `close`, `exec`, `query_count`, and
no Rows/Row surface, because that is the intersection every engine can
honour with the type system as it stands.  So the trait gives a
program engine independence, and this package's own inherent functions
— `prepare`, `step`, `query`, `table_rows` — give it rows.

**What the effect row costs, stated rather than discovered.**  The
trait's members declare `[io]`; this impl declares `[fs]`, which is
legal — a trait with no effect parameter does not pin its impls' rows,
and the standard library's own `SqliteDb` does the same.  The bill is
that a `dyn Database` call is charged the UNION over every impl in the
program (SPEC § 5.6), so a program linking this package and a network
engine pays `[fs, net]` on every `dyn` call even where it only ever
holds a file.  A concrete `SqDatabase` receiver is charged `[fs]`
alone.  `Connection` has the same obstruction recorded in its own
header and the same fix: an effect parameter on the trait.

## Two things this package does not take, and one it cannot yet

**Not btree-nv's tree.**  btree-nv describes novo-lang's own B+ tree —
a 16-byte page header, `Int` keys, `cell.encode_row` payloads — and a
SQLite page has none of that.  `sqbtree.SqPageRequest` is deliberately
its own enum; one enum covering both would be an enum whose consumer
has to know which format the page it is about is in.

**Not varint-nv.**  SQLite's varint is BIG-endian, one to nine bytes,
and its ninth byte contributes all eight of its bits rather than
seven — a shape protobuf's little-endian LEB128 has no spelling for.
Two decoders under one name would be worse than two names.

**sql-engine-nv, and it is blocked rather than declined.**  The design
takes it for the query half: SQL text in, a checked plan and an
expression evaluator out, while this package owns every byte on disk.
The dependency is not in the manifest today because `sql-engine-nv`
0.0.3 depends on `btree-nv` 0.0.2, `btree-nv` ships a module named
`cell`, `cell` is a dispatching standard library module, and the build
refuses every consumer of an assembly carrying both.  It is filed
against btree-nv; the fix is a rename and a republish in those two
repositories, and the release after it adds one line to the manifest
and changes no signature here — `sqdriver`'s prepared statement is
`SqStatement` and its values are `sqrec.SqValue`, both this package's
own types, because a driver that leaked the engine's `Statement` would
make every consumer of a SQLite file take a SQL planner with it.

## A missing row this package found

**An external-cursor feed on sql-engine-nv.**  The engine's step
protocol asks for PAGES — `SrPage(btree.PageRequest)` — and decodes
what it is handed with btree-nv's node format.  So an engine cannot be
fed a page out of a SQLite database at all, and the seam between the
query half and the storage half has to be drawn above the page: the
engine plans, and `sqbtree` walks.  What would close it is the engine
taking ROWS from a cursor it does not own, which is SQLite's own
virtual-table mechanism by another name and would serve postgres-nv's
`COPY` and redis-nv's scans as well.  It is a row for the grid, not a
gap here.

## What is scoped out of the writer, out loud

Reading a SQLite file wrong produces a wrong answer; writing one wrong
produces a file SQLite itself will not open, belonging to somebody who
has no other copy.  So `sqwrite` is a separate module and its scope is
in its own header and here:

**In:** the rollback journal in `delete` and `truncate` modes, WAL
append and the passive checkpoint, row insert/update/delete with the
b-tree splits and merges, `CREATE TABLE` and `CREATE INDEX`.

**Out, until somebody asks:** auto-vacuum and incremental vacuum (the
pointer map moves pages behind the b-tree, and a writer that ignored
it corrupts a file that reads fine), shared-cache mode, `ALTER TABLE`
beyond `ADD COLUMN`, triggers, views, foreign-key enforcement,
`VACUUM`, and the restart and truncate checkpoints.

A file using any of them is refused at open with
`SqUnsupported(feature)` carrying the feature's name, so the message
reads as a gap in this port rather than as damage to the user's data.
`sqerror.unsupported_feature` is published so a caller can collect
what a corpus asks for and turn it into the next implementation's
order of work.

## The reference implementation

libsqlite3, and its corpus is the oracle.  The format is
[documented by SQLite itself](https://sqlite.org/fileformat2.html) and
that documentation is what the module headers transcribe; the test
vectors are files `sqlite3` writes, which is the only oracle that
settles a disagreement about a format nobody else defines.

The implementation lane's first gate is therefore not a unit test: it
is a corpus of databases `sqlite3` produced — every page size, both
journal modes, a UTF-16 database, a `WITHOUT ROWID` table, a row with
a megabyte BLOB, a table that has had a column added — read back and
compared row for row with `sqlite3 -json`.

## Status

Interface only.  Nine modules, 142 public functions and three trait
members, every body a `todo()`.

- `novo pkg build` — clean, 9 modules checked.
- `novo test` — six suites, all red, every failure `not implemented`.
- `scripts/shard_audit.sh --strict` — `effect-budget`, `dep-layer`,
  `no-discharge-in-core`, `doc-examples` and `docs-pub` green; `test`
  red by design.
