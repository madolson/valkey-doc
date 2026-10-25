VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=6 nit=6

# Lists (`lists`)

https://valkey.io/topics/lists/

The page is a Redis-era tutorial with a short quick-reference section stuck on top, so the same examples show up twice and the heading hierarchy is wrong. It gets the implementation and the length limit wrong, and it leaves out the list configs and every list command added since 6.0. The most important fix: replace the "linked list" and "2^32 - 1" claims with the quicklist/listpack reality.

## Findings

### F1 [minor] INACCURATE: List length limit is not 2^32 - 1 (`topics/lists.md:396`)
**Doc says:**
> The max length of a List is 2^32 - 1 (4,294,967,295) elements.
**Problem:** The list element count is an `unsigned long` (64-bit on 64-bit builds). Nothing in the push path enforces a 2^32 - 1 cap. The real limit is memory.
**Evidence:** valkey@9.1.2:src/quicklist.h:110
```
    unsigned long count;                  /* total count of all entries in all listpacks */
```
The push and trim paths in valkey@9.1.2:src/t_list.c have no element-count check. Index arguments are parsed as `long` (`-LONG_MAX, LONG_MAX`, t_list.c:950, 1034).
**Fix:** "A list has no fixed element limit on 64-bit builds. Length is bounded by available memory." The same claim appears in topics/sets.md:157 and topics/hashes.md:115, which other reviewers should check.

### F2 [minor] INACCURATE: Lists are not plain linked lists (`topics/lists.md:7`, `topics/lists.md:103`)
**Doc says:**
> Lists are linked lists of string values.

> Lists are implemented via Linked Lists. ... Accessing an element *by index* ... requires an amount of work proportional to the index
**Problem:** Small lists are stored as a single contiguous listpack. Larger lists are a quicklist: a doubly linked list of listpack nodes, optionally LZF-compressed. The page never names the encoding or the configs that control it, and these drive memory use and index-access cost.
**Evidence:** Live 9.1.2:
```
> RPUSH bikes:repairs a
> OBJECT ENCODING bikes:repairs
listpack
> CONFIG GET list-max-listpack-size
-2
> CONFIG GET list-compress-depth
0
```
valkey@9.1.2:src/config.c:3392 `createIntConfig("list-max-listpack-size", "list-max-ziplist-size", ..., -2, ...)`, config.c:3418 `list-compress-depth ... 0`.
**Fix:** Line 7: "Lists are ordered sequences of string values." Lines 103-118: "Small lists are stored as a single compact listpack. Larger lists become a quicklist, a doubly linked list of listpack nodes. Node size is set by `list-max-listpack-size` (default -2, 8 KB per node), and `list-compress-depth` (default 0, off) compresses interior nodes. Push and pop at either end are O(1). Index access is O(N), but it walks nodes, not single elements." Link to [memory optimization](memory-optimization.md).

### F3 [minor] EXAMPLE: Missing reply, and examples assume an empty key without resetting it (`topics/lists.md:158`)
**Doc says:**
> 127.0.0.1:6379> LPUSH bikes:repairs bike:important_bike bike:very_important_bike
> 127.0.0.1:6379> LRANGE bikes:repairs 0 -1
**Problem:** Line 158 has no `(integer) 5` reply. Many blocks also reuse `bikes:repairs` but expect it to start empty. If a reader runs them in order, the replies don't match: after lines 132-142 the key holds 3 elements, so line 156 returns 6, not 3. The same carryover breaks line 81 (the key still holds `bike:1` from line 73), line 228, line 245, line 293 and line 366.
**Evidence:** Run in page order on 9.1.2:
```
> RPUSH bikes:repairs bike:1          -> 1
> RPUSH bikes:repairs bike:2          -> 2
> LPUSH bikes:repairs bike:important_bike -> 3
> RPUSH bikes:repairs bike:1 bike:2 bike:3 -> 6
> LPUSH bikes:repairs bike:important_bike bike:very_important_bike -> 8
```
**Fix:** Add `(integer) 5` after line 158. Start each self-contained block with `DEL bikes:repairs` / `(integer) 1` (or `0`), as lines 385-386 already do.

### F4 [minor] BROKEN: "Read more on the following" has nothing to read (`topics/lists.md:320`)
**Doc says:**
> We suggest that you read more on the following:
> * It is possible to build safer queues or rotating queues using `LMOVE`.
> * There is also a blocking variant of the command, called `BLMOVE`.
**Problem:** The text points readers to further reading but gives no links. The patterns it refers to are documented in the command pages.
**Evidence:** commands/lmove.md:41 `## Pattern: Reliable queue`, commands/lmove.md:64 `## Pattern: Circular list`, commands/blmove.md:13 `## Pattern: Reliable queue`.
**Fix:** "* Build reliable or circular queues with `LMOVE`: see [Reliable queue](../commands/lmove.md#pattern-reliable-queue) and [Circular list](../commands/lmove.md#pattern-circular-list). * `BLMOVE` is the blocking variant: see [BLMOVE](../commands/blmove.md)."

### F5 [minor] MISSING: No mention of LMPOP, BLMPOP, LPOS, or the COUNT argument (`topics/lists.md:13`)
**Doc says:**
> ## Basic commands ... `LPOP` removes and returns an element from the head of a list
**Problem:** The page only covers commands from the Redis 2.x era. Readers building queues won't learn that `LMPOP`/`BLMPOP` (7.0.0) pop several elements from the first non-empty of several lists, that `LPOP`/`RPOP` take a count, or that `LPOS` (6.0.6) finds an element's index.
**Evidence:** valkey@9.1.2:src/commands/lmpop.json `"since": "7.0.0"`, lpos.json `"since": "6.0.6"`. Live 9.1.2: `RPUSH x a b c` then `LPOP x 2` returns `a`, `b`.
**Fix:** Add to Basic commands: "`LPOP`/`RPOP` accept an optional count to pop several elements." "`LMPOP` and `BLMPOP` pop one or more elements from the first non-empty list among several keys." "`LPOS` returns the index of matching elements."

### F6 [minor] CONFUSING: Heading hierarchy is wrong and the same content appears twice (`topics/lists.md:91`)
**Doc says:**
> ## Examples ... ### What are Lists? ... ### First steps with Lists ... ### Capped lists
> Blocking operations on lists
> ---
**Problem:** The conceptual tutorial ("What are Lists?", "Capped lists", "Common use cases") sits as H3s under "Examples". "Blocking commands" (line 21) and "Blocking operations on lists" (line 262, a setext H2) cover the same topic. The LTRIM example at lines 81-89 is repeated word for word at lines 228-236. The queue, stack, LMOVE and LTRIM examples are explained twice.
**Evidence:** The live page renders `h2 Examples > h3 What are Lists?, h3 First steps with Lists, h3 Common use cases for lists, h3 Capped lists`, then `h2 Blocking operations on lists`.
**Fix:** Promote lines 91, 124, 191 and 212 to `##`. Convert line 262 to `## Blocking operations on lists`. Delete the duplicate LTRIM block at lines 79-89 and keep the Capped lists one. Merge "Blocking commands" (21-29) into the Basic commands list or into the Blocking operations section.

### F7 [nit] EXAMPLE: BRPOP timeout of 1 second shows 2.01s elapsed (`topics/lists.md:301`)
**Doc says:**
> 127.0.0.1:6379> BRPOP bikes:repairs 1
> (nil)
> (2.01s)
**Problem:** The timeout is 1 second, and the prose at line 306 says "return if after 1 second".
**Fix:** `(1.01s)`.

### F8 [nit] CONFUSING: Garbled sentence about capped lists (`topics/lists.md:117`)
**Doc says:**
> Another strong advantage, as you'll see in a moment, is that Lists can be taken at constant length in constant time.
**Problem:** The sentence makes no sense as written. It probably means LTRIM can cap a list.
**Fix:** "Lists can also be capped to a fixed length with `LTRIM` (see Capped lists)." Or delete it.

### F9 [nit] INACCURATE: LMOVE and BLMOVE move one element, not "elements" (`topics/lists.md:18`, `topics/lists.md:28`)
**Doc says:**
> `LMOVE` atomically moves elements from one list to another.
**Fix:** "`LMOVE` atomically pops one element from a source list and pushes it to a destination list." Same change at line 28 for `BLMOVE`.

### F10 [nit] AI-NATIVE: Commodity filler, a weak description, and a stale anecdote (`topics/lists.md:4`, `topics/lists.md:92`, `topics/lists.md:203`)
**Doc says:**
> description: > Introduction to Lists

> as the term *List* is often used in an improper way by information technology folks. For instance "Python Lists" are not what the name may suggest ... (the same data type is called Array in Ruby actually).

> The popular Twitter social network [takes the latest tweets](https://www.infoq.com/presentations/Real-Time-Delivery-Twitter)
**Problem:** The description carries no information. The Python and Ruby aside is generic and tells the reader nothing about Valkey. The Twitter link is a talk from around 2012 about a different product era. The opening paragraph doesn't say who the page is for.
**Fix:** Description: "How Valkey lists work, their commands, encoding, blocking operations, and limits." Delete lines 92-96 and start the section at line 98. Delete lines 203-204.

### F11 [nit] STYLE: Typos (`topics/lists.md:16`, `topics/lists.md:207`, `topics/lists.md:265`, `topics/lists.md:330`)
**Doc says:**
> from the tails of a list

> you want to speedup access

> a special feature that make them suitable

> It is Valkey' responsibility
**Fix:** "the tail of a list", "speed up access", "a special feature that makes them suitable", "It is Valkey's responsibility".

### F12 [nit] STYLE: Fenced code blocks have no language tag (`topics/lists.md:36`)
**Doc says:**
> ```
> 127.0.0.1:6379> LPUSH bikes:repairs bike:1
**Problem:** All 15 fenced blocks (lines 36, 48, 60, 66, 80, 131, 155, 175, 227, 244, 292, 345, 354, 365, 384) have no language tag.
**Fix:** Use ` ```bash ` or whatever tag other valkey-doc topic pages use for valkey-cli sessions, consistently.

## Unverified
- topics/lists.md:301: The exact elapsed time valkey-cli prints after a 1-second BRPOP timeout. The CLI prints timing only in interactive mode, and I didn't capture an interactive session. The 1s timeout against the 2.01s output is a mismatch on the page itself either way.
