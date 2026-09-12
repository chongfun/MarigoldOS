# Milestone 4, measured on the X3: the fingerprint is not needed

Milestone 4 asks for a `MoveFingerprint` only if measurement shows that
hashing the candidate set is too expensive. Measured on the bench X3 on
2026-09-12, from the branch at `b6c5ffc` rebased on `0fdde34`, with the
64 GB card the device has been carrying since 2026-09-10 (21 books).

## What was measured

| | |
|---|---|
| Cold catalog rebuild, 21 books | scan 454 ms, of which the identity join is 166 ms |
| The same scan's reconciliation | `matched=21 minted=0 missing=0 hashed=0 ambiguous=0 unreadable=0` |
| Boot to first paint, rebuild against a cache hit | 3321 ms against 2874 ms |
| Whole-file read and SHA-256, one continuous pass | 8,452,778 bytes in 14,585 ms, 580 kB/s |
| The same read in background slices, as shipped | 23,255 ms of wall clock, 192 kB per slice |
| Reading renders while that background read ran | layout median 16 ms, queue wait 0 ms, max 1 ms |
| Reading renders after it finished | layout median 16 ms, queue wait 0 ms, max 1 ms |
| Page turns across both captures | median 353 and 354 ms |

The captures are `storage-cache` runs from the `bench-selftest` build:
`m3-storage-1.json` (warm catalog, evidence read observed),
`m3-rebuild.json` (cold rebuild after a forced version bump),
`m3-readrate.json` (a probe build reading the whole file in one slice).

## What it says

**Reconciliation costs nothing on a card nobody reorganised.** The search
runs only between records that named no row and rows no record named, and a
scan of an unchanged shelf reports `missing=0` and `hashed=0`. Confirmed
rather than argued: the rebuild above read no book.

**A repair costs one read of the file that moved.** At 580 kB/s that is
14.6 s for an 8.45 MB book, or about 1.7 s per MB. A reader who reorganises
thirty books of that size pays around seven minutes on the next scan, once.
That is the floor for any design where the digest is the proof, and R7 says
it is: a fingerprint may narrow candidates but cannot confirm a match, so
the file that actually moved is read whatever else is added.

**A fingerprint could only skip same-length non-matches, and those barely
exist.** The search reads a file when its byte length matches that of a copy
that went missing. Byte lengths of EPUBs are spread over millions of values,
so for a library of N books the expected number of same-length pairs is
about N squared over twice the range: under one pair in a thousand-book
library. The case where lengths do collide reliably is two copies of one
book, and those share a digest as well as a length, so the search refuses
the repair as ambiguous and a sampled fingerprint would agree with the full
digest rather than rule the candidate out. Either way the read is not saved.

**So M4 closes without the fingerprint.** What would reopen it is a library
that holds many same-length books that are not copies of each other, which
would show up as `hashed` far exceeding `repaired` in the scan's bench line.
Both counters ship for that reason.

## The other thing the captures settle

The background reading of a book's bytes, which milestone 3 moved off the
first-page path, does not disturb reading. Twelve page turns landed inside
the 23 s the read was running and thirty-eight after it, and the two groups
are identical at the median and in queue wait. The read also happens once
per copy: a later capture opening the same book recorded nothing, since the
claim already said what its bytes were.

## Not measured, and why

- **A move repaired on hardware.** That needs a file renamed on the card,
  which needs the card in a computer. The path is covered by the host tests
  on the disk model, and the cost model above covers what hardware adds.
- **The join at a thousand books.** This card holds 21. The join's work is
  one ledger pass plus one 16-byte write per matched row, so the 166 ms is
  mostly per-row and extrapolates to a few seconds at a thousand books, on
  top of a rebuild that already costs about 48 s at that size. Worth
  measuring on the 1129-book card if it is still around, since an
  extrapolation across two orders of magnitude is not a measurement.
