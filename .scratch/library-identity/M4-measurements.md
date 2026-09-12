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
| A move repaired, two books renamed in a folder | `matched=19 minted=1 repaired=1 hashed=1`, join 30.1 s |
| The scan after it, nothing moved | `matched=21 repaired=0 hashed=0`, join 477 ms |

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

## A move repaired on hardware

Two books were renamed in place on a computer, both inside the folder `86`,
which holds thirteen. One had been read on the device, so the background job
had recorded its bytes in its cache claim. The other had not.

The next scan, at 21 rows:

```
sd: carried a reading place to '86/86 - Volume 01.epub'
sd: adopted 1 new book(s)
sd: found 1 book(s) again in a new place
bench: storage_ledger action=assign matched=19 minted=1 missing=1 retired=0
  duplicates=0 repaired=1 hashed=1 ambiguous=0 unreadable=0 elapsed_ms=30149
sd: catalog written, 21 epub(s)
cache: swept 1 orphan cache(s)
```

Both halves of the rule, in one scan. The book whose bytes were recorded was
found again under its own id, and the firmware carried the reader's place to
the new locator before the ledger was written. The book nobody had read was
adopted afresh, and the record of where it used to be waits as a missing
copy. One file was read to prove the one match, as the collision argument
above predicts: the second renamed book was not read, because no missing
copy with recorded bytes shared its length.

The sweep ran after the carry, not before, so the cache directory it
reclaimed was one the place had already left.

The scan that followed, on the shipping build with nothing else moved,
reports `matched=21 minted=0 repaired=0 hashed=0` and a 477 ms join: the
card has settled, and the one stale record is ageing out on the ordinary
retention.

### What the repair cost

The join took 30.1 s of a 31.5 s scan, and the read of the one file is
effectively all of it. The repaired book is the 8,452,778-byte one the bench
had been reading, which puts the scan's own read at about 285 kB/s, half the
580 kB/s the same read reaches in the background job. The scan holds the
catalog and the ledger open while it hashes, so the block cache is working
against itself; worth a look if repairs ever become common, and worth
nothing while a reorganisation is a once-a-year event.

### The gap this exposed, as designed

```
restore: no catalog match hash=8fe8850c size=8452778
```

The reading place followed the book. The record of *which book was open*
did not, because it is keyed by place, and the place changed. So the device
came up in the library rather than back in the book, and opening the book
resumes where the reader left it. That is R11 exactly, and the milestone
that moves book-open state onto the id is the reading-position work.

## Not measured, and why

- **The join at a thousand books.** This card holds 21. The join's work is
  one ledger pass plus one 16-byte write per matched row, so the 166 ms is
  mostly per-row and extrapolates to a few seconds at a thousand books, on
  top of a rebuild that already costs about 48 s at that size. Worth
  measuring on the 1129-book card if it is still around, since an
  extrapolation across two orders of magnitude is not a measurement.
