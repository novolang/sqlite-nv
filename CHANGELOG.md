# Changelog

All notable changes to sqlite-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `sqfmt` — the 100-byte database header, the four b-tree page kinds,
  the cell pointer array, SQLite's own big-endian varint, the
  local-payload split, the freelist trunk, and the two pages that hold
  no rows whatever byte 0 says.
- `sqrec` — the record format: the serial-type table with no 5 and no
  7, the three codes that occupy no bytes, one column read without the
  rest, and `pad_to` for a row written before a column was added.
- `sqbtree` — `SqPageRequest` and the cursors built on it: table scan,
  rowid seek and range, index scan and seek, the four cell layouts,
  the overflow chain a page at a time, and a depth bound.
- `sqwal` — the log header, frames whose commit record is their own
  header, the two-word checksum, the scan that builds a page-to-frame
  index, and `checkpoint_limit`.
- `sqschema` — the schema table, `is_without_rowid` and `rowid_alias`,
  and the internal index that has no `sql` to parse.
- `sqfile` — the only module that performs anything: open, a
  positional page read that consults the log first, the lock ladder at
  SQLite's own byte addresses, and `pump`.
- `sqdriver` — `SqDatabase` as the prelude's `Database`, plus prepare,
  step, query and `table_rows`.
- `sqwrite` — the commit protocol as a LIST of steps, the lock ladder
  as a function, and the row operations over them.
- `sqerror` — faults that name a page rather than a byte offset, and
  separate damage from an unsupported feature and from a lock.

### Known

- **The load-bearing interface is `sqbtree.SqPageRequest`**: the only
  way this package obtains a byte is by naming a page and being handed
  it, which is what makes reading a file somebody else wrote safe —
  out of range is answered once, by the party that knows the file's
  length.
- **`usable_size` is the page size less the reserved region**, and
  every offset check takes it as an argument.  Byte 20 is zero in
  almost every file, which is why the mistake survives every test.
- **`local_payload_bytes` keeps the last overflow page full**, and its
  modulus is the part that looks arbitrary and is not.
- **Serial codes 0, 8 and 9 occupy no bytes at all**, which is what a
  decoder ported from a tagged format gets wrong.
- **`INTEGER PRIMARY KEY` is the rowid and is NULL in the record**,
  and `WITHOUT ROWID` makes a table an index b-tree.  Both are public
  questions because missing either answers wrong rows rather than
  failing.
- **A torn WAL tail ENDS the log and is not corruption**;
  `sqwal.ends_log` exists so the opposite cannot be written by
  accident.
- **The commit order is answered as a value**, so crash safety is
  assertable with no crash, no file and no timing in the test.
- **Not btree-nv's tree and not varint-nv**: both describe a different
  format under the same word.
- **sql-engine-nv is blocked, not declined** — btree-nv 0.0.2 ships a
  module named `cell` and the build refuses every consumer of it.  The
  public surface is in this package's own types, so the release after
  the rename adds a manifest line and changes no signature.
- **No device claim**: a page is 4096 bytes and a descent holds
  several.
