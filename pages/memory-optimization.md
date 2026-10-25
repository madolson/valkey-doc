VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=4

# Memory optimization (`memory-optimization`)

https://valkey.io/topics/memory-optimization/

The page is a lightly renamed Redis-era essay. The defaults it lists are still correct, but it says hash fields cannot have a TTL, which has been false since Valkey 9.0. That is the most important fix, and the fix should also say that a field TTL costs the listpack savings.

## Findings

### F1 [minor] OUTDATED: Hash fields can have TTLs since 9.0, and a TTL removes the listpack encoding (`topics/memory-optimization.md:87`)
**Doc says:**
> However since hash fields and values are not (always) represented as full-featured Valkey objects, hash fields can't have an associated time to live (expire) like a real key, and can only contain a string. But we are okay with this, this was the intention anyway when the hash data type API was designed (we trust simplicity more than features, so nested data structures are not allowed, as expires of single fields are not allowed).

**Problem:** Valkey 9.0 added per-field expiration (HEXPIRE and related commands). This page is about memory, so the missing fact matters: a TTL on any field converts the hash from listpack to hashtable, and the hash loses the savings the page recommends.

**Evidence:** `valkey@9.1.2:src/commands/hexpire.json:5`: `"since": "9.0.0",`. `valkey@9.1.2:src/t_hash.c:375`: `if (expiry != EXPIRY_NONE || sdslen(field) > server.hash_max_listpack_value || ...` (this triggers the conversion). Live 9.1.2 run:
```
> HSET h a 1        -> (integer) 1
> OBJECT ENCODING h -> listpack
> HEXPIRE h 100 FIELDS 1 a -> 1
> OBJECT ENCODING h -> hashtable
```

**Fix:** Replace lines 87 to 91 with: "Hash fields can only hold strings. Since Valkey 9.0, individual fields can have a TTL (see [HEXPIRE](../commands/hexpire.md) and [Hashes](hashes.md#field-expiration)). Setting a TTL on any field converts the hash to a hashtable, so a hash with field TTLs does not get the compact encoding described here." Also update line 141, which says the top-level key space must support expires "so it is not practical to do this in a general way".

### F2 [minor] INACCURATE: `noeviction` has always been the default in Valkey (`topics/memory-optimization.md:181`)
**Doc says:**
> You may also want to set `maxmemory-policy` to `noeviction` (which is *not* the default value in some older versions of Valkey).

**Problem:** No Valkey release has had a different default. The parenthetical sends readers looking for a version difference that does not exist.

**Evidence:** `valkey@9.1.2:src/config.c:3365`: `createEnumConfig("maxmemory-policy", NULL, MODIFIABLE_CONFIG, maxmemory_policy_enum, server.maxmemory_policy, MAXMEMORY_NO_EVICTION, NULL, NULL),`. Live run: `CONFIG GET maxmemory-policy` returns `noeviction`.

**Fix:** "The default `maxmemory-policy` is `noeviction`. When `maxmemory` is reached, Valkey returns an out-of-memory error to write commands instead of letting the host run out of memory. For cache workloads, choose an eviction policy instead (see [Key eviction](lru-cache.md))."

### F3 [minor] MISSING: The text names Lists, but the config block leaves out `list-max-listpack-size` (`topics/memory-optimization.md:9`)
**Doc says:**
> Hashes, Lists, Sets composed of just integers, and Sorted Sets, when smaller than a given number of elements ... are encoded in a very memory-efficient way

**Problem:** Lists are listed as specially encoded, but their directive is missing from the block at lines 16 to 24. Lists also use a different sizing model: a negative value is a byte limit per node, not an entry count. A reader tuning lists has nothing to go on.

**Evidence:** `valkey@9.1.2:src/config.c:3392`: `createIntConfig("list-max-listpack-size", "list-max-ziplist-size", MODIFIABLE_CONFIG, INT_MIN, INT_MAX, server.list_max_listpack_size, -2, ...)`.

**Fix:** Add `list-max-listpack-size -2` to the block. Add one sentence: "Lists are stored as linked listpack nodes. A negative `list-max-listpack-size` limits each node by bytes (-2 means 8 KB), and a positive value limits each node by entry count." Then turn the block into a table with columns directive, default, and applies to.

### F4 [minor] MISSING: The fragmentation section never says how to measure or reduce fragmentation (`topics/memory-optimization.md:172`)
**Doc says:**
> Because of all this, the fragmentation ratio is not reliable when you had a memory usage that at the peak is much larger than the currently used memory.

**Problem:** The section explains RSS versus used memory but does not name the INFO field (`mem_fragmentation_ratio`), the diagnostic commands (`MEMORY STATS`, `MEMORY DOCTOR`, `MEMORY USAGE`), or active defragmentation. Active defragmentation is the main Valkey-specific way to get back the RSS this section describes, and it is off by default.

**Evidence:** `valkey@9.1.2:src/config.c:3300`: `createBoolConfig("activedefrag", ..., CONFIG_ACTIVE_DEFRAG_DEFAULT, ...)`. `valkey@9.1.2:src/server.h:167`: `#define CONFIG_ACTIVE_DEFRAG_DEFAULT 0`. `valkey@9.1.2:src/config.c:3397`: `active-defrag-threshold-lower` default `10`.

**Fix:** After line 177, add: "Check `mem_fragmentation_ratio` in `INFO memory`, or run [MEMORY DOCTOR](../commands/memory-doctor.md) and [MEMORY STATS](../commands/memory-stats.md). With jemalloc, the default allocator, set `activedefrag yes` to let Valkey move allocations and return fragmented memory to the OS. It is off by default and starts when fragmentation passes `active-defrag-threshold-lower` (10%)."

### F5 [nit] OUTDATED: "defaults for Valkey 7.2" label is stale (`topics/memory-optimization.md:14`)
**Doc says:**
> using the following valkey.conf directives (defaults for Valkey 7.2 are shown):

**Problem:** These values are also the 9.1.2 defaults. The 7.2 label makes readers think they may have changed.

**Evidence:** `valkey@9.1.2:src/config.c:3471-3479`: hash entries 512, set intset 512, set listpack 128/64, zset 128, hash/zset value 64.

**Fix:** "(defaults shown)".

### F6 [nit] STYLE: First-person blog voice and gendered pronoun (`topics/memory-optimization.md:64`)
**Doc says:**
> I understand the title of this section is a bit scary, but I'm going to explain in detail what this is about.

The same pattern appears at line 51 ("these new primitives"), lines 70 and 88 to 91 ("We use a trick", "we are okay with this"), lines 137 to 142 ("You may ask, why don't you do this..."), and line 144 ("so that he can pick").

**Problem:** This is the original author's essay voice. "I" is no longer a person a reader can identify, "new primitives" dates from about 2010, and "he" is not inclusive.

**Fix:** Delete line 64 and line 51's "with these new primitives". Rewrite line 144 as "Valkey makes this tradeoff explicit so users can choose the right compromise and predict how the system will behave." Replace "we" with "Valkey" throughout.

### F7 [nit] AI-NATIVE: No opening paragraph, and the savings figures are unsourced (`topics/memory-optimization.md:6`)
**Doc says:**
> uses *up to 10 times less memory* (with 5 times less memory used being the average saving).

**Problem:** The page starts directly with an H2 and never says what it covers. The 10x and 5x figures have no source. On 9.1.2, the page's own hash example uses about 3.5 times less memory than plain keys.

**Evidence:** Live 9.1.2 run, 1,000,000 objects with value `somevalue`:
```
SET object:<n>            -> used_memory:58583888
HSET object:<n[:-2]> <last 2 digits> -> used_memory:16944928 (10000 keys)
```

**Fix:** Add before line 6: "This page covers ways to reduce Valkey memory use: compact encodings for small collections and how to tune them, packing small values into hashes, bitmaps, and how the allocator reports and returns memory. It is for operators sizing instances and developers designing key schemas." Replace the figures at line 9 with "uses several times less memory (measure your own data with [MEMORY USAGE](../commands/memory-usage.md))".

### F8 [nit] AI-NATIVE: Headings that don't say what the section answers (`topics/memory-optimization.md:62`)
**Doc says:**
> ## Using hashes to abstract a very memory-efficient plain key-value store on top of Valkey

Also line 147, `## Memory allocation`, which covers maxmemory, RSS, and fragmentation.

**Problem:** The line 62 heading is long and vague, and line 64 apologizes for it. "Memory allocation" does not tell readers the section covers RSS staying high after deletes and peak provisioning.

**Fix:** Line 62: "## Packing many small string values into hashes". Line 147: "## maxmemory, RSS, and fragmentation".
