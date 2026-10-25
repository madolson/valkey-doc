VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=3 nit=3

# Hashes (`hashes`)

https://valkey.io/topics/hashes/

All examples run correctly on 9.1.2. The most important fix is the Redis-era 2^32 - 1 size limit, which Valkey's 64-bit hashtable does not have. The page also never names the listpack configs or says that fields with a TTL force hashtable encoding.

## Findings

### F1 [minor] INACCURATE: 2^32 - 1 field limit is a Redis-era figure (`topics/hashes.md:115`)
**Doc says:**
> Every hash can store up to 4,294,967,295 (2^32 - 1) field-value pairs.
> In practice, your hashes are limited only by the overall memory on the VMs hosting your Valkey deployment.

**Problem:** Valkey hashes larger than the listpack threshold use `hashtable`, which counts entries in a `size_t`. Length is returned as `unsigned long`. A 64-bit build has no 32-bit cap. The same claim appears in topics/lists.md:396 and topics/sets.md:157 and should be checked there too.
**Evidence:** `valkey@9.1.2:src/hashtable.c:310`: `size_t used[2];            /* Number of entries in each table. */`. `valkey@9.1.2:src/t_hash.c:615`: `unsigned long hashTypeLength(const robj *o) {`.
**Fix:** Replace both lines with: "The number of fields in a hash is limited only by available memory."

### F2 [minor] MISSING: small-hash encoding is vague, and the TTL interaction is undocumented (`topics/hashes.md:52`)
**Doc says:**
> It is worth noting that small hashes (i.e., a few elements with small values) are
> encoded in special way in memory that make them very memory efficient.

**Problem:** "A few elements with small values" gives readers no thresholds to tune. The page also does not say that setting a TTL on any field converts the hash to `hashtable` encoding. That cost matters to anyone who adopts field expiration on many small hashes.
**Evidence:** 9.1.2 run:
```
CONFIG GET hash-max-*  -> hash-max-listpack-entries 512, hash-max-listpack-value 64
HSET bike:1 ...; OBJECT ENCODING bike:1 -> listpack
HSETEX h EX 100 FIELDS 1 a 1; OBJECT ENCODING h -> hashtable
```
`valkey@9.1.2:src/t_hash.c:375`: `if (expiry != EXPIRY_NONE || sdslen(field) > server.hash_max_listpack_value || sdslen(value) > server.hash_max_listpack_value)` (this condition converts the hash to hashtable).
**Fix:** Replace the paragraph with: "Hashes with at most `hash-max-listpack-entries` fields (default 512), where every field and value is at most `hash-max-listpack-value` bytes (default 64), use the compact `listpack` encoding. Above either limit, or as soon as any field has an expiration time, the hash is converted to `hashtable` encoding. See [Memory optimization](memory-optimization.md)."

### F3 [minor] BROKEN: HPEXPIRETIME link is labeled HEXPIRETIME (`topics/hashes.md:98`)
**Doc says:**
> [`HEXPIRETIME`](../commands/hexpiretime.md), [`HEXPIRETIME`](../commands/hpexpiretime.md), ...

**Problem:** The second link points to hpexpiretime, but its label is HEXPIRETIME. Readers see the same command twice, and HPEXPIRETIME does not appear anywhere on the page.
**Fix:** Change the second label to `` [`HPEXPIRETIME`](../commands/hpexpiretime.md) ``.

### F4 [nit] AI-NATIVE: weak description and untagged code fences (`topics/hashes.md:4`)
**Doc says:**
> description: > Introduction to Hashes

**Problem:** The description restates the title. All four fenced blocks (lines 10, 35, 45, 67) have no language tag.
**Fix:** Set the description to "Valkey hashes: field-value records, common commands, per-field expiration (9.0+), encoding, and performance." Tag the four fences as `text` (or `bash` to match other topics).

### F5 [nit] STYLE: typos and grammar in the field expiration section and the intro (`topics/hashes.md:28`)
**Doc says:**
> L87 "it's corresponding value", "after the time has past", L91 "some some hash commands", L53 "encoded in special way ... that make them", L100 "while also associate", L28 "actually the number of fields", L109 "Most Hash commands"

**Problem:** Typos and filler.
**Fix:** Change to "its corresponding value", "after the time has passed", "some hash commands", "encoded in a special way ... that makes them", "while also associating", delete "actually", and use "Most hash commands".

### F6 [nit] CONFUSING: HRANDFIELD limitation is described as an "ability" (`topics/hashes.md:92`)
**Doc says:**
> Another side effect of using volatile fields (fields with time to live) in hash objects, is the ability to randomly choose from hash objects.

**Problem:** The sentence calls a limitation an ability. Read literally, it says expiration adds random selection. The behavior is real: on 9.1.2, `HLEN` returned 1 and `HRANDFIELD` returned empty for a hash whose fields had all logically expired.
**Fix:** "Volatile fields also affect random selection: when most fields of a large hash have logically expired, `HRANDFIELD` might not find a live field and can return an empty reply."

## Unverified
- topics/hashes.md:89: "expired hash fields are deleted via a periodic job" may omit lazy deletion on access. In the 9.1.2 run, HLEN dropped from 1 to 0 after HGETALL/HRANDFIELD, but active expire could not be disabled (DEBUG is off), so the cause is ambiguous. `t_hash.c:461` mentions lazy expiry only for overwrites.
- topics/hashes.md:104: "some commands which override hash fields (e.g. `HSET` and `HMSET`)" does not list the full set. Confirmed that HSET clears the TTL (HTTL -1 after HSET) and HSETEX KEEPTTL keeps it. HINCRBY, HSETNX, and HINCRBYFLOAT were not tested.
