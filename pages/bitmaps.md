VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=5 nit=3

# Bitmaps (`bitmaps`)

https://valkey.io/topics/bitmaps/

Short page whose examples run correctly on 9.1.2. The prose has drifted from the commands, though. It gets the SETBIT argument order wrong, treats the 512 MB default as a hard limit, says ranges are byte-only (BIT ranges arrived in 7.0), and leaves out BITFIELD and SETBIT's allocation latency. Fix the SETBIT argument description first.

## Findings

### F1 [minor] INACCURATE: SETBIT argument order is wrong (`topics/bitmaps.md:58`)
**Doc says:**
> The `SETBIT` command takes as its first argument the bit number, and as its second argument the value to set the bit to, which is 1 or 0.
**Problem:** The first argument is the key, the second is the offset, and the third is the value. A reader who follows the prose gets an error.
**Evidence:** The page's own example at line 36 uses `SETBIT pings:2024-01-01-00:00 123 1`. When the offset is put in the key position on 9.1.2:
```
$ valkey-cli -p 20140 SETBIT k 1 bitnum
ERR bit is not an integer or out of range
```
**Fix:** "`SETBIT key offset value` sets the bit at `offset` to `value` (0 or 1). If the offset is past the end of the string, the string grows to fit and the new bytes are zero-filled."

### F2 [minor] INACCURATE: 512 MB is a configurable default, not a fixed string maximum (`topics/bitmaps.md:9`)
**Doc says:**
> Since strings are binary safe blobs and their maximum length is 512 MB, they are suitable to set up to 2^32 different bits.

and line 56: "of 4 billion users using just 512 MB of memory."
**Problem:** The bit-offset limit comes from `proto-max-bulk-len`, which defaults to 512 MB and can be raised. The 512 MB / 2^32 figure is the default, not a fixed maximum.
**Evidence:** valkey@9.1.2:src/config.c:3461 `createLongLongConfig("proto-max-bulk-len", ..., 1024 * 1024, LONG_MAX, server.proto_max_bulk_len, 512ll * 1024 * 1024, ...)`. valkey@9.1.2:src/bitops.c:592-593:
```
/* Limit offset to server.proto_max_bulk_len (512MB in bytes by default) */
if (loffset < 0 || (!mustObeyClient(c) && (loffset >> 3) >= server.proto_max_bulk_len)) {
```
Default run: `SETBIT k 4294967296 1` returns `ERR bit offset is not an integer or out of range`, and `SETBIT k 4294967295 0` succeeds (`STRLEN` 536870912).
**Fix:** "A bitmap can address up to 2^32 bits (512 MB) by default. The limit is set by `proto-max-bulk-len`."
Tracked: valkey-io/valkey-doc#97

### F3 [minor] OUTDATED: BITCOUNT and BITPOS ranges are described as byte-only (`topics/bitmaps.md:73`)
**Doc says:**
> Both `BITPOS` and `BITCOUNT` are able to operate with byte ranges of the string
**Problem:** Since 7.0.0 both commands take a `BYTE | BIT` unit, so ranges can be given in bits.
**Evidence:** valkey@9.1.2:src/commands/bitcount.json and bitpos.json mark the unit argument `"since": "7.0.0"`. On 9.1.2, `BITPOS pings:2024-01-01-00:00 1 0 -1 BIT` returns `123`.
**Fix:** "`BITPOS` and `BITCOUNT` can be limited to a range of the string, given in bytes (default) or in bits with the `BIT` modifier."

### F4 [minor] MISSING: BITFIELD and BITFIELD_RO left out of the bit-group command list (`topics/bitmaps.md:67`)
**Doc says:**
> There are three commands operating on group of bits:
**Problem:** `BITFIELD` and `BITFIELD_RO` are in the bitmap command group, but the page doesn't mention them or link to the Bitfields topic page. Readers storing packed counters won't find them from here.
**Evidence:** valkey@9.1.2:src/commands/bitfield.json and bitfield_ro.json: `"group": "bitmap"`. topics/bitfields.md exists.
**Fix:** Add a fourth item: "`BITFIELD` reads, sets, and increments integer fields of arbitrary width inside the string. See [Bitfields](bitfields.md)."

### F5 [minor] MISSING: Performance section skips BITCOUNT/BITPOS and SETBIT allocation latency (`topics/bitmaps.md:103`)
**Doc says:**
> `SETBIT` and `GETBIT` are O(1).
> `BITOP` is O(n), ...

and line 47: "constant-time single bit operations, like setting a bit to 1 or 0"
**Problem:** The section doesn't give complexity for BITCOUNT or BITPOS (both O(N)). It also omits the main operational risk on this page: a SETBIT to a high offset on a new or short key allocates the whole string at once and can block the server for hundreds of milliseconds.
**Evidence:** valkey@9.1.2:src/commands/bitcount.json and bitpos.json: `"complexity": "O(N)"`. commands/setbit.md:14-19: "Valkey needs to allocate all intermediate memory which can block the server for some time ... setting bit number 2^32 -1 (512MB allocation) takes ~300ms".
**Fix:** Add: "`BITCOUNT` and `BITPOS` are O(N) in the size of the range scanned. A `SETBIT` past the end of a string allocates and zero-fills every byte up to that offset in one step. A first write near 2^32 allocates 512 MB and can block the server. See [SETBIT](../commands/setbit.md)."

### F6 [nit] CONFUSING: Longest-streak use case is buried in the "Bit Operations" section with no commands (`topics/bitmaps.md:81`)
**Doc says:**
> For example imagine you want to know the longest streak of daily visits ... with a few `BITPOS` calls, or simply fetching and analyzing the bitmap client-side, it is possible to easily compute the longest streak.
**Problem:** "For example" follows a BITCOUNT example about sensor pings, not streaks, so it reads as a continuation of the wrong example. No commands are shown for the streak computation, and the key-splitting paragraph (lines 94-99) sits in the same section without a heading.
**Fix:** Move lines 81-92 under `## Example: daily visit streaks`, with one `SETBIT` and one `BITCOUNT` call. Move lines 94-99 under `## Splitting large bitmaps across keys`.

### F7 [nit] AI-NATIVE: Weak description and a generic "Example" heading (`topics/bitmaps.md:4`)
**Doc says:**
> description: > Introduction to Bitmaps

and line 26: `## Example`
**Problem:** The description adds nothing to the title. "Example" doesn't say which problem the section solves.
**Fix:** description: "Use bit operations on Valkey strings for compact sets and flags: SETBIT, GETBIT, BITCOUNT, BITPOS, BITOP, size limits, and performance." Heading: `## Example: tracking hourly sensor pings`.

### F8 [nit] EXAMPLE: Fenced code blocks have no language tag (`topics/bitmaps.md:35`)
**Doc says:**
> ```
> 127.0.0.1:6379> SETBIT pings:2024-01-01-00:00 123 1
**Problem:** The blocks at lines 35 and 76 have no language tag. Their output matches 9.1.2.
**Fix:** Tag both blocks the way other valkey-doc pages tag valkey-cli sessions (for example `text`).
