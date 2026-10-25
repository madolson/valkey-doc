VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=7 nit=6

# Secondary indexing (`indexing`)

https://valkey.io/topics/indexing/

The techniques are correct, but this is the antirez-era page almost unchanged. It never mentions Valkey Search, which is now the native way to get secondary indexes, and it ignores cluster mode, where its multi-key update patterns fail. Fix first: add a section near the top pointing to topics/search.md, and say when hand-built indexes are still the right choice.

## Findings

### F1 [minor] MISSING: No mention of Valkey Search, and the page still says native indexing is a future feature (`topics/indexing.md:7`)
**Doc says:**
> It is fair to say that, natively, Valkey only offers *primary key access*.

and at line 689:
> While non immediately trivial this is a very useful indexing strategy that in the future may be implemented in Valkey in a native way.
**Problem:** valkey-doc already documents the Valkey Search module (FT.CREATE, numeric, tag, full-text and vector indexes over hashes and JSON). A reader who arrives at "Secondary indexing" is never told that it exists, so they hand-build indexes that the module maintains automatically.
**Evidence:** topics/search.md:5-11: "Valkey-Search allows users to create indexes and perform searches, incorporating complex filters. Users can index data using either Valkey Hash or Valkey-JSON data types." topics/indexing.md has no link to search.md or valkey-bundle.md.
**Fix:** After line 19, add: "If you can load modules, [Valkey Search](search.md) creates and maintains numeric, tag, full-text and vector indexes over hashes and JSON documents for you. The techniques on this page use only core data structures. Use them when modules are unavailable, or when you need an index shape that Valkey Search does not provide." Replace line 689-690 with "This strategy is not built into Valkey; the complexity can be encapsulated in a client library."

### F2 [minor] MISSING: Multi-key index updates fail in cluster mode, and the page never says so (`topics/indexing.md:120`)
**Doc says:**
> The operation may be wrapped in a `MULTI`/`EXEC` transaction in order to make sure both fields are updated or none.

and lines 460-463 (`MULTI` / `ZADD myindex ...` / `HSET index.content ...` / `EXEC`), and line 279 (Lua script for the update).
**Problem:** Every pattern on the page keeps the index in one key and the data in other keys (`user:1` plus `user.age.index`, `myindex` plus `index.content`). In cluster mode those keys hash to different slots, so the MULTI/EXEC transaction and the Lua script are rejected. Readers on a cluster find this out in production.
**Evidence:** valkey@9.1.2:src/cluster.c:1323: `addReplyError(c, "-CROSSSLOT Keys in request don't hash to the same slot");`
**Fix:** Add after line 121: "In cluster mode, a transaction or script can only touch keys in one hash slot. Put the index and the objects it covers under one hash tag, for example `{user}:1` and `{user}.age.index`, or update them in separate commands and accept that they can briefly disagree. A single hash tag also puts the whole index on one shard."

### F3 [minor] INACCURATE: MULTI/EXEC does not give "both or none" (`topics/indexing.md:120`)
**Doc says:**
> The operation may be wrapped in a `MULTI`/`EXEC` transaction in order to make sure both fields are updated or none.
**Problem:** Valkey transactions have no rollback. If one command fails at runtime, the others still run. What MULTI/EXEC guarantees is isolation: no other client sees the hash updated without the index.
**Evidence:** topics/transactions.md:169-172: "Valkey does not support rollbacks of transactions since supporting rollbacks would have a significant impact on the simplicity and performance of Valkey."
**Fix:** "Wrap both commands in a `MULTI`/`EXEC` transaction so that they execute together and no other client sees one updated without the other. Transactions do not roll back, see [Transactions](transactions.md)."

### F4 [minor] INACCURATE: A prefix lookup with `LIMIT 0 1` returns the next entry when the key is missing (`topics/indexing.md:354`)
**Doc says:**
> And search for the key with:
>
>     ZRANGE myindex [mykey: + BYLEX LIMIT 0 1
>     1) "mykey:myvalue"

and line 271: "This will return the single entry of `banana` if it exists."
**Problem:** The range is open-ended (`+`). If `mykey` is absent, the query returns whatever entry sorts next, so a reader who follows "extract the part after the colon" gets another key's value.
**Evidence:** 9.1.2 run, with only `other:1` in the set:
```
$ valkey-cli -p 20540 ZRANGE myindex '[mykey:' + BYLEX LIMIT 0 1
other:1
```
**Fix:** Bound the range: `ZRANGE myindex "[mykey:" "[mykey:\xff" BYLEX LIMIT 0 1`, or add: "If the key is absent this returns the next entry in order, so check that the result starts with `mykey:` before using it." Apply the same change at line 268.

### F5 [minor] EXAMPLE: Hexastore example output is not in lexicographic order (`topics/indexing.md:505`)
**Doc says:**
>     1) "spo:antirez:is-friend-of:matteocollina"
>     2) "spo:antirez:is-friend-of:wonderwoman"
>     3) "spo:antirez:is-friend-of:spiderman"

and lines 513-515: `is-friend-of`, `was-at-conference-with`, `talked-with`.
**Problem:** The section's whole premise is that BYLEX returns results sorted by byte value. Both outputs show orders that Valkey cannot produce, which undercuts the explanation.
**Evidence:** 9.1.2 run (commands sent through stdin so that `\xff` is parsed):
```
spo:antirez:is-friend-of:matteocollina
spo:antirez:is-friend-of:spiderman
spo:antirez:is-friend-of:wonderwoman
sop:antirez:matteocollina:is-friend-of
sop:antirez:matteocollina:talked-with
sop:antirez:matteocollina:was-at-conference-with
```
**Fix:** Swap lines 506/507 and lines 514/515 to match the output above.

### F6 [minor] EXAMPLE: Composite index range drops products priced exactly at the upper bound (`topics/indexing.md:435`)
**Doc says:**
>     ZRANGE myindex [0056:0010.00 [0056:0030.00 BYLEX
**Problem:** Entries carry a `:product_id` suffix, so `0056:0030.00:91` sorts after `[0056:0030.00` and is excluded. The prose says "a price between 10 and 30 dollars". A $30.00 product silently goes missing.
**Evidence:** 9.1.2 run with `0056:0028.44:90` and `0056:0030.00:91` in the set:
```
$ valkey-cli -p 20540 ZRANGE myindex '[0056:0010.00' '[0056:0030.00' BYLEX
0056:0028.44:90
```
**Fix:** `ZRANGE myindex "[0056:0010.00" "[0056:0030.00:\xff" BYLEX`, plus one sentence: "Because entries carry trailing fields, end an inclusive upper bound with the separator and `\xff`."

### F7 [minor] CONFUSING: The purge example shows a lookup query, not a completion query (`topics/indexing.md:290`)
**Doc says:**
> When we actually query the index in order to complete the user input, we may see something like that:
>
>     ZRANGE myindex "[banana:" + BYLEX LIMIT 0 10
>     1) "banana:123"
>     2) "banaooo:1"
>     3) "banned user:49"
**Problem:** The completion query defined at line 239 is `[prefix` to `[prefix\xff`. This query is the open-ended `banana:` lookup instead, and its results (`banned user`) do not share a typed prefix. The reader has to work out that the user typed `ban`.
**Evidence:** Line 239: `ZRANGE myindex "[bit" "[bit\xff" BYLEX`. The doc's own output for line 290 includes entries that do not start with `banana:`.
**Fix:** `ZRANGE myindex "[ban" "[ban\xff" BYLEX LIMIT 0 10`, and change the lead-in to "If the user typed `ban`, we may see:".

### F8 [nit] EXAMPLE: `ZREM` example passes a stray `0` as a member (`topics/indexing.md:275`)
**Doc says:**
>     ZREM myindex 0 banana:1
**Problem:** `ZREM` takes members, not a score. This also tries to delete a member named `0`.
**Evidence:** valkey@9.1.2:src/commands/zrem.json has arguments `key` and `member` (multiple). On 9.1.2, `ZREM myindex 0 banana:1` returns `1` because `0` is treated as a nonexistent member.
**Fix:** `ZREM myindex banana:1`.

### F9 [nit] CONFUSING: `\xff` only works inside valkey-cli's own parser, not as a shell argument (`topics/indexing.md:239`)
**Doc says:**
>     ZRANGE myindex "[bit" "[bit\xff" BYLEX
**Problem:** Pasted as `valkey-cli ZRANGE ...` from a shell, `\xff` arrives as four literal characters and the query returns nothing. The escape works only in the interactive prompt or through stdin. Client libraries need a real 0xFF byte. Lines 504 and 512 have the same problem.
**Evidence:** 9.1.2: running the line 504 query as shell arguments returned an empty list. The same command sent through stdin returned the three `spo:` entries.
**Fix:** After line 242, add: "`\xff` is valkey-cli syntax for the byte 0xFF. In client code, append the byte itself."

### F10 [nit] STYLE: Typos and grammar errors throughout
**Doc says:** line 130 "they it is often possible"; 170 "in a lexicographically fashion"; 205 "This prefixes are mandatory"; 279 "should be send"; 301 "remove the item from the list" (it is a sorted set); 317 "One simple way do deal with this issues"; 492 "In can add"; 622 "0s ad 1s"; 689 "non immediately trivial"; 69 "objects IDs".
**Problem:** These are errors a reader trips on.
**Evidence:** Quoted above.
**Fix:** "then it is"; "in a lexicographical fashion"; "These prefixes"; "should be sent"; "from the index"; "One simple way to deal with these issues"; "I can add"; "0s and 1s"; "not immediately trivial"; "object IDs".

### F11 [nit] STYLE: Section headings render as H1, so they are peers of the page title, and code blocks have no language tag (`topics/indexing.md:21`)
**Doc says:**
> Simple numerical indexes with sorted sets
> ===
**Problem:** The live page has `<h1 class="page-title">` followed by ten more `<h1>` sections (setext `===`). Every example except the Ruby one is an indented code block, so none of them carries a language tag.
**Evidence:** `curl -sL https://valkey.io/topics/indexing/` shows `<h1 id="simple-numerical-indexes-with-sorted-sets">`, `<h1 id="lexicographical-indexes">`, and so on.
**Fix:** Convert `===` sections to `##` and `---` subsections to `###`. Convert the indented examples to fenced blocks tagged with a language (for example `text` or `bash`).

### F12 [nit] DUPLICATE: The 128-bit big-endian index idea appears on both pages (`topics/indexing.md:403`)
**Doc says:**
> An alternative approach is just to store numbers, for example 128 bit integers, directly in their binary form. However for this to work, you need to store the numbers in *big endian format*
**Problem:** topics/sorted-sets.md:174-182 explains the same technique ("index elements by a 128-bit unsigned integer ... the 128 bit number in big endian").
**Evidence:** topics/sorted-sets.md:177-178: "with a 16 byte prefix consisting of **the 128 bit number in big endian**".
**Fix:** Keep the explanation here, since indexing.md is canonical. Cut sorted-sets.md:174-182 to one sentence linking to [Secondary indexing](indexing.md#using-numbers-in-binary-form).

### F13 [nit] AI-NATIVE: The front matter description and the opening paragraph do not say what the reader will learn (`topics/indexing.md:4`)
**Doc says:**
> description: >
>     Building secondary indexes in Valkey

and line 7: "Valkey is not exactly a key-value store, since values can be complex data structures. However it has an external key-value shell..."
**Problem:** The description repeats the title. The opening sentence is philosophy, and the page's scope (which structures, which query shapes) only arrives in line 9.
**Fix:** Description: "Build numeric, lexicographic, composite, graph, and multi-dimensional secondary indexes in Valkey with sorted sets, sets, and lists." Open line 7 with: "Valkey looks up data by key name. This page shows how to build secondary indexes yourself with sorted sets, sets, and lists, for when you need to query by another field."
