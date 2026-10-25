VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=3 nit=4

# Bitfields (`bitfields`)

https://valkey.io/topics/bitfields/

Short page, example runs correctly on 9.1.2. Most important fix: the supported width range is stated backwards (signed goes to 64 bits, unsigned to 63), and the page recommends bitfields for counters without mentioning that overflow silently wraps by default.

## Findings

### F1 [minor] INACCURATE: Width range is wrong (`topics/bitfields.md:8`)
**Doc says:**
> For example, you can operate on anything from unsigned 1-bit integers to signed 63-bit integers.
**Problem:** Signed integers go up to 64 bits (`i64`). Unsigned integers go up to 63 bits (`u63`). `u64` is rejected. The sentence implies signed tops out at 63.
**Evidence:** Valkey 9.1.2:
```
> BITFIELD x GET i64 0
0
> BITFIELD x GET u64 0
ERR Invalid bitfield type. Use something like i16 u8. Note that u64 is not supported but i64 is.
> BITFIELD x GET u63 0
0
```
**Fix:** "Signed integers can be 1 to 64 bits wide (`i1` to `i64`). Unsigned integers can be 1 to 63 bits wide (`u1` to `u63`)."

### F2 [minor] MISSING: Overflow behavior not mentioned for a page that pitches counters (`topics/bitfields.md:11`)
**Doc says:**
> Bitfields support atomic read, write and increment operations, making them a good choice for managing counters and similar numerical values.
**Problem:** Counters overflow silently by default (WRAP). A reader using this page alone will not learn that `OVERFLOW SAT` or `OVERFLOW FAIL` exist. The example itself does `INCRBY u32 #0 -50`, which would wrap to a huge value if the price ever went below 0.
**Evidence:** Valkey 9.1.2:
```
> BITFIELD x INCRBY u8 0 300
44
```
commands/bitfield.md:25-27 documents `OVERFLOW [WRAP|SAT|FAIL]`.
**Fix:** Add after line 11: "Increments that overflow wrap around by default. Use the `OVERFLOW` subcommand to saturate (`SAT`) or reject the operation (`FAIL`) instead. See [`BITFIELD`](../commands/bitfield.md)."

### F3 [minor] CONFUSING: `#` offset notation unexplained and prose calls `#1` "offset 1" (`topics/bitfields.md:24`)
**Doc says:**
> Bike 1 initially costs 1,000 (counter in offset 0) ... and it now has an owner (offset 1).
**Problem:** The example uses `#0` and `#1`, which are field indexes multiplied by the type width, so `u32 #1` is bit offset 32. The prose calls it "offset 1", which in BITFIELD syntax (`u32 1`) means bit offset 1 and would overlap the price counter. The `#` prefix is never explained.
**Evidence:** commands/bitfield.md explains `#` offsets; the topic page does not. `STRLEN bike:1:stats` after the example returns `8`, i.e. two 32-bit fields at bit offsets 0 and 32.
**Fix:** Replace "(counter in offset 0)" with "(the first 32-bit field, `#0`)" and "(offset 1)" with "(the second 32-bit field, `#1`)". Add one sentence before the example: "A `#` prefix on the offset means the field index, multiplied by the type width: `u32 #1` starts at bit 32."

### F4 [nit] STYLE: Typos in example prose (`topics/bitfields.md:22`, `topics/bitfields.md:24`)
**Doc says:**
> a 32-bit wide bitfield per for each bike.

> the number of owners does as well.Finally, you can look
**Problem:** "per for" is a duplicated word. Missing space after the period renders as "well.Finally" on the live page. The single-item bullet on line 24 is a narrative paragraph, not a list.
**Fix:** "a 32-bit wide bitfield for each bike." / "does as well. Finally, you can look". Drop the leading `* ` on line 24.

### F5 [nit] MISSING: Commands not linked, `BITFIELD_RO` purpose unstated (`topics/bitfields.md:16-17`)
**Doc says:**
> * `BITFIELD` atomically sets, increments and reads one or more values.
> * `BITFIELD_RO` is a read-only variant of `BITFIELD`.
**Problem:** Neither command links to its reference page. The reader is not told why `BITFIELD_RO` exists (safe to run on replicas and in read-only scripts).
**Fix:** "* [`BITFIELD`](../commands/bitfield.md) atomically sets, increments, and reads one or more values. * [`BITFIELD_RO`](../commands/bitfield_ro.md) accepts only `GET`, so it can run on replicas and in read-only contexts."

### F6 [nit] EXAMPLE: Fenced block has no language tag (`topics/bitfields.md:26`)
**Doc says:**
> ```
**Problem:** Renders as `data-lang="plain"` on the live page.
**Fix:** Tag it ```` ```text ````. bitmaps.md:35 has the same gap.

### F7 [nit] AI-NATIVE: Weak description and generic headings (`topics/bitfields.md:4`, `topics/bitfields.md:14`, `topics/bitfields.md:20`)
**Doc says:**
> description: > Introduction to Bitfields

> ## Basic commands

> ## Example
**Problem:** The description restates the title. "Example" does not say what the example shows.
**Fix:** description: "Store and atomically update multiple fixed-width integer counters inside one Valkey string with BITFIELD." Headings: "## Bitfield commands", "## Example: per-bike price and owner counters".
