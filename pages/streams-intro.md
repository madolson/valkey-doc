VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=9 nit=10

# Streams (`streams-intro`)

https://valkey.io/topics/streams-intro/

This is the Redis 5/6-era tutorial with names swapped to Valkey. Several examples no longer match what 9.1.2 prints, and the special-ID recap tells readers to use `$` with `XREADGROUP`, which errors. The most important fix: correct the XAUTOCLAIM and XINFO examples and the special-ID recap so they match 9.1.2.

## Findings

### F1 [minor] EXAMPLE: XAUTOCLAIM example shows a 2-element reply and an impossible cursor sequence (`topics/streams-intro.md:644`)
**Doc says:**
> 127.0.0.1:6379> XAUTOCLAIM race:italy italy_riders Lora 60000 (1692632662819-0 COUNT 1
> 1) "1692632662819-0"
> 2) 1) 1) "1692632647899-0"

**Problem:** Since 7.0, XAUTOCLAIM returns three elements: the cursor, the claimed entries, and the IDs of deleted entries it removed from the PEL. The page shows two at L645-648 and L656-659, and the prose at L651 describes only two. The second call also can't happen as shown. It starts the scan exclusively after `1692632662819-0`, the highest ID in the PEL, so it can't return the lower ID `1692632647899-0` with a non-zero cursor.
**Evidence:** `valkey@9.1.2:src/commands/xautoclaim.json` history: `"7.0.0", "Added an element to the reply array, containing deleted entries the command cleared from the PEL"`. 9.1.2 output, using the page's data:
```
> XAUTOCLAIM race:italy italy_riders Alice 0 0-0 COUNT 1
1) "1692632662819-0"
2) 1) 1) "1692632647899-0"
      2) 1) "rider"
         2) "Royce"
3) (empty array)
> XAUTOCLAIM race:italy italy_riders Lora 0 (1692632662819-0 COUNT 1
1) "0-0"
2) (empty array)
3) (empty array)
```
**Fix:** Regenerate both examples on 9.1.2, using the returned cursor as the next `start`. Add the third element to each reply. Change L651 to: "Like `XCLAIM`, the command replies with the claimed messages. It also returns a cursor for the next call, and a list of IDs that were in the PEL but have since been deleted from the stream (those are removed from the PEL)."

### F2 [minor] INACCURATE: special-ID recap says to use `$` with XREADGROUP (`topics/streams-intro.md:830`)
**Doc says:**
> So for instance if I want only new entries with `XREADGROUP` I use this ID to signify I already have all the existing entries

**Problem:** `$` is rejected by XREADGROUP. New entries for a group come from `>`. `$` belongs with `XREAD` and `XGROUP CREATE`/`SETID`.
**Evidence:** 9.1.2:
```
> XREADGROUP GROUP italy_riders Bob STREAMS race:italy $
ERR The $ ID is meaningless in the context of XREADGROUP: you want to read the history of this consumer by specifying a proper ID, or use the > ID to get new messages. The $ ID would just return an empty result set.
```
**Fix:** Replace with "So for instance if I want only new entries with `XREAD` I use this ID ...", and add: "`XREADGROUP` rejects `$`; use `>` instead."

### F3 [minor] EXAMPLE: XINFO STREAM, GROUPS, CONSUMERS and HELP outputs are pre-7.0 (`topics/streams-intro.md:680`)
**Doc says:** XINFO STREAM (L680-699) lists 14 fields. XINFO GROUPS (L704-713) ends at `last-delivered-id`. XINFO CONSUMERS (L720-739) shows only `name`, `pending`, `idle`. XINFO HELP (L752) prints `Prints this help.`
**Problem:** 9.1.2 also returns `max-deleted-entry-id`, `entries-added` and `recorded-first-entry-id` (STREAM), `entries-read` and `lag` (GROUPS), and `inactive` (CONSUMERS). These are the fields readers need to measure consumer lag. HELP prints `Print this help.`
**Evidence:** `valkey@9.1.2:src/commands/xinfo-groups.json` history `"7.0.0", "Added the entries-read and lag fields"`. `xinfo-consumers.json` history `"7.2.0", "Added the inactive field, and changed the meaning of idle"`. 9.1.2 output:
```
 9) "max-deleted-entry-id"
10) "0-0"
11) "entries-added"
12) (integer) 5
13) "recorded-first-entry-id"
...
    9) "entries-read"
   10) (integer) 3
   11) "lag"
   12) (integer) 2
...
   7) "inactive"
   8) (integer) 7
...
9)     Print this help.
```
**Fix:** Regenerate all four outputs on 9.1.2. Add one sentence after L713: "`lag` is the number of entries not yet delivered to the group, which is the usual consumer-backlog metric." Add one after L739: "`idle` is the time since the consumer's last attempted interaction, `inactive` the time since its last successful one."

### F4 [minor] INACCURATE: complexity claims contradict each other and the command metadata (`topics/streams-intro.md:73`)
**Doc says:**
> L7: random access in O(1) time
> L73: Accessing any single entry is O(n), where _n_ is the length of the ID.
> L260: `XRANGE` complexity is *O(log(N))* to seek, and then *O(M)* to return M elements

**Problem:** The page gives three different costs for the same operation. "Random access in O(1)" and "O(n) in the ID length" are not statements a reader can act on.
**Evidence:** `valkey@9.1.2:src/commands/xrange.json`: `"complexity": "O(N) with N being the number of elements being returned. If N is constant (e.g. always asking for the first 10 elements with COUNT), you can consider it O(1)."`
**Fix:** L7: "These include fast lookup of any entry by ID and complex consumption strategies, such as consumer groups." Replace L72-78 with: "`XADD` is O(1). Fetching an entry or range by ID with `XRANGE` is O(N) in the number of entries returned, so it is effectively constant for small `COUNT`. Streams are implemented as a radix tree of listpack nodes. See each command page for exact complexity." Make L260 consistent with this.

### F5 [minor] OUTDATED: page says time-based retention is impossible, then introduces MINID as XTRIM-only (`topics/streams-intro.md:794`)
**Doc says:**
> There is currently no option to tell the stream to just retain items that are not older than a given period

> L818: `XTRIM` is designed to accept different trimming strategies. Another trimming strategy is **MINID**

**Problem:** `MINID` (since 6.2) evicts entries with IDs below a given value. Auto-generated IDs are millisecond timestamps, so `MINID <now_ms - period>` is exactly time-based retention. `XADD` also accepts `MINID` and `LIMIT`, so this isn't XTRIM-only.
**Evidence:** `valkey@9.1.2:src/commands/xadd.json` history: `"6.2.0", "Added the NOMKSTREAM option, MINID trimming strategy and the LIMIT option."` `xtrim.json` history: `"6.2.0", "Added the MINID trimming strategy and the LIMIT option."`
**Fix:** Replace L794 from "There is currently no option" through the end of the paragraph with: "To keep only recent entries, trim by ID instead of length. `XADD key MINID ~ <ms> * ...` or `XTRIM key MINID ~ <ms>` evicts entries whose ID is below `<ms>`. Because auto-generated IDs start with a millisecond timestamp, passing the current time minus the retention period gives time-based retention. Use `LIMIT` to cap how many entries one call evicts." Change L818 to say both `XADD` and `XTRIM` accept `MAXLEN` and `MINID`.

### F6 [minor] CONFUSING: prose and Ruby example use `mygroup`/`my_stream_key`, examples use `italy_riders`/`race:italy` (`topics/streams-intro.md:427`)
**Doc says:**
> L427: I want to read from the stream using the consumer group `mygroup`
> L471: is reading via the same group `mygroup`
> L492: GroupName = "mygroup"  /  L513: :my_stream_key

**Problem:** The commands the prose describes use `italy_riders` on `race:italy`. The Ruby consumer reads a group the page never creates, so running it as written fails with `NOGROUP`. It also uses `redis-rb`, but the Valkey client page lists Valkey GLIDE for Ruby (`clients/ruby/valkey-GLIDE.json`).
**Evidence:** L420 `XREADGROUP GROUP italy_riders Alice ...`. L461 `XREADGROUP GROUP italy_riders Bob ...`.
**Fix:** Replace `mygroup` with `italy_riders` at L427 and L471. In the Ruby code set `GroupName = "italy_riders"`, replace `:my_stream_key` with `'race:italy'`, and add before L481: "This uses the redis-rb client, which works against Valkey. Create the group first with `XGROUP CREATE race:italy italy_riders $ MKSTREAM`."

### F7 [minor] MISSING: XREAD `+` (read the last entry) is not mentioned, including in the special-ID recap (`topics/streams-intro.md:838`)
**Doc says:**
> So we have `-`, `+`, `$`, `>` and `*`, and all have a different meaning

**Problem:** Since Valkey 8.0.0, `XREAD ... STREAMS key +` returns the last entry in the stream. That is the common "get latest" call, and the recap at L828 describes `+` only as a range bound. The page also doesn't cover the `<ms>-*` form at L836, where it says `*` is "auto select an ID".
**Evidence:** `valkey` commit a8e745117 "Xread last entry in stream (#7388) (#13117)", first tagged in 8.0.0. `valkey@9.1.2:src/t_stream.c:2290`: `} else if (strcmp(objectGetVal(c->argv[i]), "+") == 0) {`. 9.1.2:
```
> XREAD STREAMS race:italy +
1) 1) "race:italy"
   2) 1) 1) "1692632678249-0"
         2) 1) "rider"
            2) "Norem"
```
**Fix:** After L832 add: "With `XREAD`, `+` returns the last entry in the stream (Valkey 8.0 and later). `XREADGROUP` rejects `+`." At L836 add: "`<ms>-*` sets the milliseconds part and auto-generates the sequence."

### F8 [minor] MISSING: the macro-node size configs behind `~`, XDEL, and memory claims are never named (`topics/streams-intro.md:796`)
**Doc says:**
> Altering the single macro node, consisting of a few tens of elements, is not optimal.

**Problem:** L796, L802 and L872 all depend on node size: approximate trimming granularity, and memory that isn't reclaimed after `XDEL`. The page doesn't give the defaults, and no topic page in valkey-doc mentions these configs.
**Evidence:** `valkey@9.1.2:src/config.c:3462`: `createLongLongConfig("stream-node-max-entries", ..., 100, ...)`. `src/config.c:3478`: `createSizeTConfig("stream-node-max-bytes", ..., 4096, MEMORY_CONFIG, ...)`. `grep -rn stream-node-max topics/` returns nothing.
**Fix:** Replace "consisting of a few tens of elements" with "holding up to `stream-node-max-entries` entries (default 100) or `stream-node-max-bytes` bytes (default 4096)". At L872 add: "Memory is freed per node, so the reclaim granularity is set by these two configs."

### F9 [minor] AI-NATIVE: undated, unsourced throughput and latency numbers (`topics/streams-intro.md:882`)
**Doc says:**
> L882: `XADD` is very fast and can easily insert from half a million to one million items per second in an average machine if pipelining is used.
> L898-919: "Latency tests results" ... "The system used for this benchmark is very slow compared to today's standards."

**Problem:** The numbers have no version, hardware, client, or date. They come from a Redis 5-era Ruby test and say nothing about Valkey 9.x, which has I/O threading. A reader can't reproduce or quote them.
**Evidence:** The page itself gives no version or hardware. The test was a Ruby harness (L900) on unspecified hardware (L919).
**Fix:** Delete the "Latency tests results" section (L898-919) and the throughput sentence at L882. Keep the push-model explanation (L886-896), and replace the cut text with: "To measure on your hardware, run `valkey-benchmark -t xadd` or time your own producer and consumer, and pin the Valkey version you tested."

### F10 [nit] OUTDATED: Redis OSS version gates on features every Valkey release has (`topics/streams-intro.md:129`)
**Doc says:**
> L129: If you're running Redis OSS 7 or later, you can also provide an explicit ID consisting of the milliseconds part only.
> L630: The `XAUTOCLAIM` command, added in Redis OSS 6.2

**Problem:** Every Valkey release supports both features, so the Redis version gate only confuses Valkey readers.
**Evidence:** `valkey@9.1.2:src/commands/xadd.json` history `"7.0.0", "Added support for the <ms>-* explicit ID form."`. `xautoclaim.json` `"since": "6.2.0"`. Valkey forked from 7.2.4.
**Fix:** L129: "You can also provide an explicit ID consisting of the milliseconds part only." L630: "The `XAUTOCLAIM` command implements the claiming process described above."

### F11 [nit] INACCURATE: `-` is 0-0, not "basically 0-1" (`topics/streams-intro.md:828`)
**Doc says:**
> the smallest ID possible (that is basically `0-1`)

**Evidence:** `valkey@9.1.2:src/t_stream.c:1904-1906`: `if (buf[0] == '-' && buf[1] == '\0') { id->ms = 0; id->seq = 0;`
**Fix:** "the smallest ID possible (`0-0`)".

### F12 [nit] OUTDATED: speculative future-feature sentence (`topics/streams-intro.md:822`)
**Doc says:**
> Another useful eviction strategy that may be added to `XTRIM` in the future, is to remove by a range of IDs

**Problem:** The roadmap note is years old, and `MINID` already covers moving data out by ID.
**Fix:** Delete L822.

### F13 [nit] DUPLICATE: XREAD blocking, `$`, and `~` trimming are re-explained in depth on command pages
**Doc says:** L317-329 (XREAD BLOCK, `$`, FIFO fairness), L796-802 (`MAXLEN ~` semantics).
**Problem:** `commands/xread.md:121-186` ("Blocking for data", "The special `$` ID", "How multiple clients blocked on a single stream are served") and `commands/xtrim.md:30-42` / `commands/xadd.md:58-68` explain the same mechanics in parallel, so the copies can drift. They already disagree on node size ("a few tens" here, "a few tens more" in xadd.md).
**Fix:** Keep the tutorial narrative here and make the command pages the reference for options. At L802 link to `../commands/xtrim.md` for `~` and `LIMIT`. At L329 link to `../commands/xread.md#the-special--id`.

### F14 [nit] INACCURATE: `MKSTREAM` called a subcommand; `ENTRIESREAD` not mentioned (`topics/streams-intro.md:395`)
**Doc says:**
> using the optional `MKSTREAM` subcommand as the last argument

**Evidence:** `valkey@9.1.2:src/commands/xgroup-create.json` lists `mkstream` and `entriesread` as arguments of `XGROUP CREATE`.
**Fix:** "using the optional `MKSTREAM` argument".

### F15 [nit] MISSING: persistence section omits WAITAOF (`topics/streams-intro.md:846`)
**Doc says:**
> AOF must be used with a strong fsync policy if persistence of messages is important in your application.

**Problem:** `WAITAOF` (documented in `commands/waitaof.md`) lets a producer confirm fsync per write without setting `appendfsync always` globally.
**Fix:** Add a bullet: "`WAITAOF` blocks until preceding writes are fsynced to the AOF on the primary and/or a number of replicas. See [WAITAOF](../commands/waitaof.md)."

### F16 [nit] BROKEN: `<key>` in bold prose is emitted as a raw HTML tag (`topics/streams-intro.md:677`)
**Doc says:**
> For instance **XINFO STREAM <key>** reports information about the stream itself.

**Evidence:** Live HTML: `For instance <strong>XINFO STREAM <key><`. The browser treats `<key>` as an unknown tag and drops it.
**Fix:** "For instance `XINFO STREAM <key>` reports information about the stream itself."

### F17 [nit] STYLE: first-person author voice throughout (`topics/streams-intro.md:349`)
**Doc says:**
> L349: so I decided to keep Kafka's (TM) terminology

Same pattern at L190, L205, L229, L386, L427, L641, L652, L802, L830.
**Problem:** "I" is the original Redis author. In Valkey docs it's unattributed and reads as if Valkey made the decision.
**Fix:** Use "you" or "we" for the reader's actions. L349: "Valkey keeps Kafka's (TM) terminology because Kafka popularized the idea."

### F18 [nit] STYLE: code fences have no language tag and prompts are inconsistent
**Doc says:** Every fence except the Ruby block (L483) has no language, for example L32, L87, L105, L339, L361. L320 and L744 use a `>` prompt, while the rest use `127.0.0.1:6379>`.
**Fix:** Tag CLI sessions as ```` ```bash ```` (or the site's convention for valkey-cli), and tag diagrams and signatures as ```` ```text ````. Use `127.0.0.1:6379>` at L320 and L744.

### F19 [nit] AI-NATIVE: weak description and a front section that duplicates the tutorial below it (`topics/streams-intro.md:4`)
**Doc says:**
> description: Introduction to Streams

L20-78 ("Basic commands", "Examples", "Performance") repeat XADD/XRANGE/XREAD examples that L81-331 cover again ("Streams basics").
**Fix:** description: "How Valkey streams work: entry IDs, range queries, blocking reads, consumer groups, claiming, trimming, and persistence guarantees." Fold "Examples" into "Streams basics" and rename "Performance" to "Time complexity of stream operations" (see F4).

## Unverified
- topics/streams-intro.md:802: "the Python client defaults to approximate". Didn't check valkey-py or valkey-glide Python defaults within the timebox. The claim names no client, so even if true it should name the library and version or be cut.
- topics/streams-intro.md:872: memory isn't reclaimed until a macro node is completely empty. Consistent with node-level freeing in t_stream.c, but I didn't measure it on 9.1.2. Unstable adds `XDELEX`/`XACKDEL` (`origin/unstable:src/commands/xdelex.json`), which are unreleased and out of scope for 9.1.2.
