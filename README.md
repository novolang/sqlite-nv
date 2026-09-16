# sqlite-nv

A SQLite database is a whole relational database held in one ordinary
file, and it is the most widely exchanged binary format there is: a
phone backup, a browser profile, a mail store, a firmware dump, an
attachment somebody sent you. The format is specified by SQLite itself
in [The SQLite Database File Format](https://sqlite.org/fileformat2.html).
This package reads and writes that format natively, with no
`libsqlite3` anywhere in the build, so a program that wants to open a
database file does not need a C toolchain to do it.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What it is

A database file is a sequence of **pages**, all the same size, numbered
from 1. The first 100 bytes of page 1 are the **database header**, and
every other number in the file is computed from what is in it.

A table is a **b-tree**: an ordered structure spread over pages, where
the pages above hold keys and the addresses of the pages below, and the
pages at the bottom hold the data. Finding a row is a **descent**: read
the root page, compare the key, read the child, and repeat until a
bottom page. A table b-tree is keyed by the **rowid**, a 64-bit integer
SQLite gives every row. An index b-tree is keyed by the indexed columns
instead.

| Byte 0 of the page | The page is | It holds |
| --- | --- | --- |
| 2 | an interior index page | child pointers and index keys |
| 5 | an interior table page | child pointers and rowids |
| 10 | a leaf index page | index keys, with the rowid inside them |
| 13 | a leaf table page | rowids and records |

A row on a leaf page is a **record**: a header of types followed by a
body of values, and nothing in the body says where a value ends. The
header is a varint giving its own length, then one **serial type** per
column, and the length of every value is a function of its serial type
alone.

| Serial code | The value is | Bytes it occupies |
| --- | --- | --- |
| 0 | NULL | 0 |
| 1 to 6 | a big-endian two's-complement integer | 1, 2, 3, 4, 6, 8 — there is no 5 and no 7 |
| 7 | an IEEE-754 binary64 | 8 |
| 8 | the integer 0 | 0 |
| 9 | the integer 1 | 0 |
| 10, 11 | reserved, and no file uses them | — |
| even, 12 and up | a BLOB | (N − 12) / 2 |
| odd, 13 and up | text, in the database's encoding | (N − 13) / 2 |

Every database has one table that describes the others, the **schema
table**, rooted at page 1. It has five columns — the kind of object, its
name, the table it belongs to, the page its b-tree is rooted at, and
the `CREATE` statement as it was typed — and it is read by the same
cursor as any other table.

A database may be in **WAL mode**, in which case a second file,
`<name>-wal`, holds pages that have been changed and not yet copied
back. A **frame** is a 24-byte header and one page of content, a
**commit** is a frame whose header carries the database's new size in
pages, and a **checkpoint** copies committed frames back into the
database file. A read in WAL mode consults the log first, then the file.

| Structure | Size |
| --- | --- |
| Database header, at the front of page 1 | 100 bytes |
| B-tree page header, on a leaf | 8 bytes |
| B-tree page header, on an interior page | 12 bytes |
| Page 1's b-tree page header begins at | byte 100 |
| Page size | a power of two from 512 to 65536 |
| Usable size | the page size less the reserved region |
| WAL header | 32 bytes |
| WAL frame header, followed by one page | 24 bytes |
| A varint | big-endian, 1 to 9 bytes |

**Nothing in this package reads a byte of a database by itself.** A
cursor names a page number and waits: `sqbtree.step` answers a row, an
index entry, the end of the walk, a failure, or a request, and the
caller answers a request with `sqbtree.feed_page`. `sqfile` is the only
module that opens a file, and `sqfile.pump` is that loop written once.

The reason is the input. The realistic file this package is handed is
one somebody else wrote, and possibly wrote on purpose. A reader that
held a buffer and an offset would have to check every offset it
computes and get every one of them right. A reader that can only say
*give me page 47* has one place where out-of-range is answered:
`sqfile` knows the file's length and the cursor does not, so it answers
`SqNoSuchPage` once rather than re-checking at every decode site.

The other consequence is that a caller can put the file anywhere. A
database inside a tar archive, in object storage, in a memory image or
behind a network block device is read by answering `sqbtree`'s requests
from there. `sqfile` is a convenience over the requests and not the
only door.

## Install

```
novo pkg add sqlite-nv
```

## Example

```novo
use std.list
use std.str
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

// One row as text, its values separated by a bar.
fn render(row: [sqrec.SqValue]) -> Str
    str.join(list.map(row, sqrec.show), " | ")

fn main() [io, fs]
    for line in dump("app.db", "users")
        println(line)
```

Nothing in that mentions a page.

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: sqlite-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `sqfmt` | The database header, the four b-tree page kinds and their headers, the cell pointer array, SQLite's varint, the usable size, the local-payload split, the freelist trunk and the two pages that hold no rows. |
| `sqrec` | The record format: the serial types, one column read without the rest, the padding a row written before a column was added needs, and the encoder. |
| `sqbtree` | The page request and the cursors built on it: table scan, rowid seek and range, index scan and seek, the four cell layouts, the overflow chain a page at a time, and the depth bound. |
| `sqwal` | The write-ahead log: its header, its frames, the two-word checksum, the salt rule, the scan that builds a page-to-frame index, and the limit a checkpoint may copy to. |
| `sqschema` | The schema table: its rows, the table and index definitions they become, the `WITHOUT ROWID` and rowid-alias questions, and the internal index that has no SQL. |
| `sqfile` | The only module that performs anything: open, close, a page read that consults the log first, the lock ladder, the catalogue and the cursor loop. |
| `sqdriver` | SQL over a file: the prelude's `Database` trait, prepare, step, reset, query, and every row of one table by name. |
| `sqwrite` | Writing: the journal modes, the commit order as a list of steps, the lock ladder as a function, and the row operations over them. |
| `sqerror` | The faults, which name a page rather than a byte offset, and the three questions a caller asks of one. |

Every module except `sqfile`, `sqdriver` and `sqwrite` declares no
effects at all: it is arithmetic over bytes the caller already holds,
so the whole format is testable with no file in the test.

## How to choose an entry point

**`sqdriver` is SQL and a `dyn Database`.** `SqDatabase` implements the
prelude's `Database` trait, the standard library's one contract for
engines, so a program written against `dyn Database` runs over a SQLite
file this package opened, over the standard library's own `SqliteDb`
and over another engine's client, with nothing in it naming any of the
three. `sqdriver.query` and `sqdriver.table_rows` are the two calls
most first uses will be.

**`sqfile.rows_of` reads a table with no SQL at all.** It is what a
backup reader, a forensic tool or a migration checker wants, and it
works even when a table's `CREATE` statement uses something the parser
does not accept: the rows are in the b-tree whether or not the
statement that made them parses.

**`sqbtree.step` and `sqbtree.feed_page` are the primitives.** Use them
when the pages do not come from a file on this machine, when the caller
wants to batch or interleave reads, or when a walk has to be put away
and resumed. `sqfile.pump` is the same loop for the ordinary case.

**`sqfmt` and `sqrec` are the format without any walk.** A tool that
inspects a page it already holds, or decodes one record, needs those
two and nothing else. Neither can touch a file.

## The rules a user needs

1. **The usable size is the page size less the reserved region, and
   every offset is checked against it** (file format section 1.3). Byte
   20 of the header is zero in almost every file anybody has, which is
   why a reader that used the page size instead passes every test it is
   given and then reads a payload short by up to 255 bytes on the first
   encrypted or checksummed database it meets. `sqfmt.usable_size` is
   the number, and every check in this package takes it as an argument.
2. **A payload that does not fit a page is split, and the split keeps
   the last overflow page full** (section 1.6).
   `sqfmt.local_payload_bytes` is that arithmetic. Its modulus is the
   part that looks arbitrary and is not: a reader that dropped it takes
   the right number of bytes off the page and then starts the chain at
   the wrong offset, which produces a record that decodes into the
   wrong values.
3. **An overflow page is four bytes of next-page pointer and then
   payload, with nothing saying how much of the last page is used.**
   The payload length in the cell is the only thing that ends the
   chain, so a reader that walked until the pointer was zero appends
   whatever was in the last page before. `sqbtree.overflow_part` takes
   what is still owed and answers this page's share.
4. **Page 1's b-tree page header begins at byte 100, and its cell
   offsets are still measured from the start of the page**
   (section 1.2). `sqfmt.page_header_offset` is that off-by-100,
   written once.
5. **Serial codes 0, 8 and 9 occupy no bytes at all** (section 2.1).
   Codes 8 and 9 carry the integers 0 and 1 in the type itself. A
   decoder that gave every integer a width reads the next column's
   bytes as this one's, and every column after it is shifted.
   `sqrec.serial_bytes` is a table and not a subtraction, because there
   is no 5-byte and no 7-byte integer.
6. **Text is in the database's encoding, not the column's**
   (section 1.3, byte 56). There is no per-value encoding byte, so
   `sqrec.decode_record` takes the encoding as an argument. A reader
   that assumed UTF-8 answers mojibake on a UTF-16 database rather than
   an error.
7. **A record may hold fewer columns than its table, and that is not
   damage.** `ALTER TABLE … ADD COLUMN` does not rewrite the rows
   already stored, and a column past the end of a record reads that
   column's DEFAULT. `sqrec.pad_to` finishes the row and takes the
   defaults as an argument.
8. **An `INTEGER PRIMARY KEY` column is an alias for the rowid, and the
   record holds NULL in its place.** A reader that handed the NULL back
   answers NULL for every primary key in the table.
   `sqschema.rowid_alias` is the question, and `sqfile.rows_of` substitutes the
   rowid so a caller cannot skip it.
9. **A `WITHOUT ROWID` table is an index b-tree.** Walking one with a
   table cursor decodes index cells as table cells and produces rowids
   made out of payload lengths. `sqschema.is_without_rowid` is the
   question, and `sqfile.catalogue` asks it for every table it finds.
10. **An index SQLite created for a `UNIQUE` or `PRIMARY KEY`
    constraint has no `sql`.** It is named `sqlite_autoindex_<table>_<n>`
    and its columns come from the table's constraint instead, so a
    reader that required `sql` drops exactly the indexes a query would
    most like to use.
11. **`sqrec.compare` implements the BINARY collation only.** A column
    declared `COLLATE NOCASE` or `COLLATE RTRIM` is one this reader
    orders wrong. `sqschema.non_binary_collations` lists a table's
    such columns, and an index on one can be scanned but not seeked
    correctly.
12. **In WAL mode the log holds the newest version of a page, and it is
    consulted first on every read.** A reader that opened only the
    database file answers rows as of the last checkpoint, which is not
    an error anywhere and is simply an old answer. `sqwal.frame_for` is
    the lookup and `sqfile.read_page` performs it.
13. **A frame commits when its header's database-size field is
    non-zero. There is no separate commit record.**
    `sqwal.frame_commits` is that test.
14. **A frame whose salts do not match the log header's is from before
    the last checkpoint.** The log is written in place and never
    truncated, so a forward scan walks over such frames and stops.
    `sqwal.frame_is_current` is the comparison.
15. **A checksum that does not continue the running total ends the log
    rather than condemning it.** The last frame of a log whose writer
    was killed mid-write is a frame that was never committed, and
    dropping it is the whole of crash recovery. A recovery that called
    it corruption throws away every committed transaction in the log.
    `sqwal.ends_log` is the line between the two.
16. **The WAL checksum is not a CRC.** It is two 32-bit words folded
    over the input eight bytes at a time. The log header's magic says
    which endianness the checksums are in; every other field in the WAL
    is big-endian whichever magic is written.
17. **A checkpoint may copy back only as far as the oldest frame any
    open reader is still reading at.** Copying past it changes a page
    under a reader that is mid-query, which does not fail — it answers
    a row from two transactions at once. `sqwal.checkpoint_limit`
    answers the number rather than performing the copy.
18. **A descent is bounded.** A corrupt interior page whose child
    pointer names an ancestor makes a descent read pages that decode,
    in an order that never terminates. `sqbtree.depth_limit` is the
    bound, and it is a function of the page count because a b-tree over
    N pages cannot be deeper than N.
19. **A page number outside `1..page_count` is `SqNoSuchPage`, and page
    0 does not exist.** The header's page count is trustworthy only
    when its `version_valid_for` field equals its change counter;
    otherwise the file's length divided by the page size is what
    decides, and `sqfile` falls back to that.
20. **A database past a gigabyte has one page that holds no rows
    whatever byte 0 says.** SQLite's lock ranges live at byte
    0x40000000, and the page covering them is not a b-tree node.
    `sqfmt.lock_byte_page` is which page that is, and a reader that
    forgot is correct on every fixture anybody writes.
21. **A file using a feature this port does not read is refused at open
    with `SqUnsupported`, carrying the feature's name.** That is a
    different answer from corruption: it says this reader has not built
    that yet, rather than that the user's database is damaged.
    `sqerror.unsupported_feature` is how a caller collects what a
    corpus asks for.
22. **The commit order is answered as a value.** `sqwrite.commit_steps`
    returns the steps a commit has to take, in order, rather than
    performing them, so a test can assert that the journal is synced
    before the first page is written — the crash-safety property,
    stated as an assertion, with no crash in the test.
    `sqwrite.next_lock` is the lock ladder in the same shape.
23. **A transaction's changed pages are held in memory until commit**,
    so a transaction's size is bounded by the process's memory.
24. **A handle is a value and a cursor is a value.** `read_page` does
    not change the handle and `step` returns a new cursor, so two
    cursors over one file cannot alias and a join may hold an outer
    walk and an inner seek with no lock (SPEC section 14). A caller
    that wants two independent readers opens the file twice.

## The `std.sql` driver

The prelude's `Database` trait is narrow on purpose: `close`, `exec`
and `query_count`, with no rows and no transaction type, because that
is the intersection every engine can honour with the type system as it
stands. So the trait gives a program engine independence, and this
package's own functions — `prepare`, `step`, `query`, `query_row` and
`table_rows` — give it rows.

The trait's members declare `[io]` and this implementation declares
`[fs]`, which is legal: a trait with no effect parameter does not pin
its implementations' effect rows, and the standard library's own
`SqliteDb` does the same. The bill is that a `dyn Database` call is
charged the union over every implementation in the program (SPEC
section 5.6), so a program linking this package and a network engine
pays `[fs, net]` on every `dyn` call even where it only ever holds a
file. A concrete `SqDatabase` receiver is charged `[fs]` alone.

A `DbError` from the trait has five variants and a message; a
`sqerror.SqFault` names a page, an offset and a feature.
`sqdriver.last_fault` answers the second, which is the only place
"page 4192 did not decode" survives the narrower contract.

## What is not included

- **A query planner, joins and indexes at query time.** A prepared
  statement in this release reads one table; a statement over more than
  one is refused. `sqfile.rows_of` and the cursors are what a program
  composes its own answer from.
- **A dependency on a SQL engine.** The design takes
  [sql-engine-nv](https://novo-lang.org/packages/sql-engine-nv) for the
  query half — SQL text in, a checked plan and an expression evaluator
  out — while this package owns every byte on disk. It is not in the
  manifest of this release. Nothing in the public surface changes when
  it lands: `sqdriver`'s prepared statement is `SqStatement` and its
  values are `sqrec.SqValue`, both this package's own types, because a
  driver that exposed an engine's `Statement` would make every consumer
  of a SQLite file take a SQL planner with it.
- **Auto-vacuum and incremental vacuum.** A pointer map moves pages
  behind the b-tree, and a reader that ignored it reads a page that has
  been relocated. `sqfmt.has_pointer_map` is the question, asked at
  open, and such a file is refused with `SqUnsupported`.
- **`VACUUM`, `ALTER TABLE` beyond `ADD COLUMN`, triggers, views and
  foreign-key enforcement.** Those are a query engine's work rather
  than a file format's, except `VACUUM`, which rewrites the whole file
  and wants a design of its own.
- **The `restart` and `truncate` checkpoints.** Both need a registry of
  open readers that this package does not have. `SqPassive` and
  `SqFull` are the two modes the first implementation performs.
- **The `-shm` wal-index.** It is a shared-memory hash table over the
  frames, it is not part of the format's compatibility promise, and it
  is rebuilt by any process that finds it missing. This package scans
  the log instead, which costs one pass and buys not reproducing a
  structure whose layout may change between SQLite releases. A reader
  that has to share an index with other processes needs the `-shm`
  file, and this release does not have it.
- **Shared-cache mode**, and any form of multi-process writing beyond
  the lock protocol.
- **A microcontroller build.** A page is 4096 bytes by default and a
  descent holds several of them; the package builds for the system and
  application tiers and claims no device.

## Related packages

- [btree-nv](https://novo-lang.org/packages/btree-nv) is novo-lang's
  own B+ tree: a 16-byte page header, `Int` keys and its own row
  encoding. A SQLite page has none of that, which is why
  `sqbtree.SqPageRequest` is its own enum rather than btree-nv's — one
  enum covering both would be an enum whose consumer has to know which
  format the page it is about is in.
- [pager-nv](https://novo-lang.org/packages/pager-nv) is the file and
  the write-ahead log for that tree, in a format of its own. Take it
  for a database this program owns; take this package for a file
  another program wrote.
- [sql-engine-nv](https://novo-lang.org/packages/sql-engine-nv) is the
  SQL lexer, parser, planner and executor. Its cursors ask for pages in
  btree-nv's format, so it cannot be fed a page out of a SQLite
  database: the seam between the query half and the storage half has to
  be drawn above the page, with the engine planning and `sqbtree`
  walking.
- [varint-nv](https://novo-lang.org/packages/varint-nv) is protobuf's
  varint, which is little-endian LEB128. SQLite's is big-endian, one to
  nine bytes, and its ninth byte contributes all eight of its bits
  rather than seven. `sqfmt.decode_varint` is the second shape, and two
  decoders under one name would be worse than two names.
- `std.sql` in the standard library declares the `Database` trait, and
  its `SqliteDb` is the same format reached through a `sqlite3`
  subprocess. Take that when a C SQLite is present and its query engine
  is what is wanted; take this package to read the file directly.

## Tests

```bash
novo test --isolate tests/sqfmt_tests.nv    # 11 tests: the header and the offsets
novo test --isolate tests/sqrec_tests.nv    #  8 tests: the serial types and the record
novo test --isolate tests/sqbtree_tests.nv  #  6 tests: the cells, the cursors and the chain
novo test --isolate tests/sqwal_tests.nv    #  8 tests: the frames, recovery and the checkpoint
novo test --isolate tests/sqschema_tests.nv #  8 tests: the schema table and its two questions
novo test --isolate tests/sqhost_tests.nv   #  7 tests: the file, the driver and the commit order
```

The reference is libsqlite3, and a corpus of files `sqlite3` itself
wrote is the oracle: it is the only thing that settles a disagreement
about a format nobody else defines. The implementation's first gate is
therefore not a unit test but a corpus — every page size, both journal
modes, a UTF-16 database, a `WITHOUT ROWID` table, a row with a
megabyte BLOB, and a table that has had a column added — read back and
compared row for row against `sqlite3 -json`.

The first five suites above hold 41 tests between them and not one file
handle, which is what keeping the format modules free of effects buys.
Only `sqhost_tests.nv` declares `[fs]`.

The tests compile today and fail at run, each on the
`not implemented: sqlite-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

Nine modules, 143 public functions and three trait members, every body
a `todo()`.

| Item | Implemented |
| --- | --- |
| `sqfmt.magic`, `.header_size`, `.decode_header`, `.encode_header`, `.new_header` | no |
| `sqfmt.usable_size`, `.is_legal_page_size`, `.with_reserved`, `.page_offset` | no |
| `sqfmt.page_header_offset`, `.has_pointer_map`, `.pointer_map_page` | no |
| `sqfmt.lock_byte_page`, `.pending_byte` | no |
| `sqfmt.decode_page_header`, `.encode_page_header`, `.page_header_bytes`, `.page_header` | no |
| `sqfmt.kind_of`, `.kind_byte`, `.is_leaf`, `.is_table`, `.cell_pointers`, `.free_bytes` | no |
| `sqfmt.decode_varint`, `.encode_varint`, `.varint_len` | no |
| `sqfmt.local_payload_bytes`, `.max_local_payload`, `.overflow_page_count`, `.overflow_payload_bytes` | no |
| `sqfmt.decode_free_trunk`, `.free_trunk_capacity` | no |
| `sqrec.serial_of`, `.serial_code`, `.serial_bytes`, `.serial_for` | no |
| `sqrec.decode_record`, `.decode_header`, `.column_at`, `.pad_to` | no |
| `sqrec.encode_record`, `.encoded_size`, `.type_name` | no |
| `sqrec.as_int`, `.as_real`, `.as_text`, `.as_blob`, `.compare`, `.show` | no |
| `sqbtree.scan_table`, `.seek_rowid`, `.range_rowid`, `.scan_index`, `.seek_index` | no |
| `sqbtree.step`, `.feed_page`, `.feed_alloc`, `.awaiting`, `.depth_limit` | no |
| `sqbtree.table_leaf_cell`, `.table_interior_cell`, `.index_cell`, `.overflow_part` | no |
| `sqwal.header_size`, `.frame_header_size`, `.decode_header`, `.encode_header`, `.new_header` | no |
| `sqwal.frame_offset`, `.frame_capacity`, `.decode_frame`, `.encode_frame` | no |
| `sqwal.checksum_seed`, `.checksum_zero`, `.frame`, `.checksum` | no |
| `sqwal.frame_is_current`, `.frame_commits`, `.ends_log`, `.frame_for` | no |
| `sqwal.index_new`, `.index_add`, `.checkpoint_limit`, `.may_restart` | no |
| `sqschema.schema_root`, `.schema_name`, `.row_of_record` | no |
| `sqschema.to_table_defs`, `.to_index_defs`, `.table_def`, `.column_def` | no |
| `sqschema.find_table`, `.indexes_on`, `.is_without_rowid`, `.rowid_alias` | no |
| `sqschema.is_internal_name`, `.non_binary_collations` | no |
| `sqfile.open`, `.open_read_only`, `.close`, `.header`, `.is_stale` | no |
| `sqfile.read_page`, `.pump`, `.catalogue`, `.rows_of`, `.scan_wal` | no |
| `sqfile.lock`, `.unlock`, `.lock_level` | no |
| `sqdriver.close`, `.exec`, `.query_count` (the `Database` members) | no |
| `sqdriver.open`, `.open_read_only`, `.last_fault`, `.catalogue` | no |
| `sqdriver.prepare`, `.columns`, `.step`, `.reset` | no |
| `sqdriver.query`, `.query_row`, `.table_rows`, `.table_names`, `.table_def` | no |
| `sqwrite.commit_steps`, `.rollback_steps`, `.next_lock`, `.journal_path`, `.wal_path` | no |
| `sqwrite.begin`, `.stage_page`, `.commit`, `.rollback`, `.recover`, `.checkpoint` | no |
| `sqwrite.insert_row`, `.update_row`, `.delete_row` | no |
| `sqerror.describe`, `.is_corruption`, `.page_of`, `.unsupported_feature` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
