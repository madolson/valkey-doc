VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=5 nit=4

# Sets (`sets`)

https://valkey.io/topics/sets/

The page is accurate at the command level but its examples contradict each other when run in order, the stated size limit is Redis-era and wrong for Valkey's hashtable, and it omits the encodings and multi-set complexity that are the Valkey-specific value. Fix the example outputs first: a reader who runs the page top to bottom gets different results.

## Findings

### F1 [minor] EXAMPLE: Example outputs assume fresh keys but the page reuses the same keys (`topics/sets.md:66`)
**Doc says:**
> 127.0.0.1:6379> SADD bikes:racing:france bike:1 bike:2 bike:3
> (integer) 3
**Problem:** The Examples section (lines 29-35) already creates `bikes:racing:france` {1,2,3} and `bikes:racing:usa` {1,4}. Every later block re-adds the same members and shows counts as if the keys were empty. Same pattern at lines 93-94 (`(integer) 2`), 109-112 (`3`, `2`), and 141-142 (`5`).
**Evidence:** Running the page in order on 9.1.2:
```
> SADD bikes:racing:france bike:1 bike:2 bike:3
0
> SADD bikes:racing:usa bike:1 bike:4
0
> SADD bikes:racing:france bike:1 bike:2 bike:3 bike:4 bike:5
2
```
**Fix:** Start the Tutorial section with `127.0.0.1:6379> DEL bikes:racing:france bikes:racing:usa bikes:racing:italy` / `(integer) 2` and drop the redundant re-`SADD` lines at 93-94 and 109-112, and put `DEL bikes:racing:france` before line 141. Or state once above the first block: "Each example starts from an empty database."

### F2 [minor] CONFUSING: SINTER described as producing a list of all bikes in every race (`topics/sets.md:101`)
**Doc says:**
> For instance we may want a list of all the bikes racing in France, the USA, and some other races. We can do this using the `SINTER` command, which performs the intersection
**Problem:** "All the bikes racing in France, the USA, and some other races" reads as a union. `SINTER` returns only bikes racing in every race, so the prose and the example (`1) "bike:1"`) disagree.
**Evidence:** Example at line 115-116 returns only `bike:1`. `SUNION` at 117-121 is what returns "all the bikes".
**Fix:** "For instance, we may want the bikes that race in France, the USA, and Italy. `SINTER` returns the intersection of the sets. `SUNION` returns every bike in any of them, and `SDIFF` returns the bikes in the first set that are in none of the others:"

### F3 [minor] OUTDATED: 2^32 - 1 member limit is a Redis-era figure (`topics/sets.md:157`)
**Doc says:**
> The max size of a Set is 2^32 - 1 (4,294,967,295) members.
**Problem:** Large sets use the `hashtable` encoding, whose size is a `size_t` on 64-bit builds. No 2^32 cap exists in 9.1.2. The only 32-bit counter is the intset length, and intsets convert to another encoding past `set-max-intset-entries` (512).
**Evidence:** valkey@9.1.2:src/t_set.c:473-476
```
unsigned long setTypeSize(const robj *subject) {
    if (subject->encoding == OBJ_ENCODING_HASHTABLE) {
        return hashtableSize((const hashtable *)objectGetVal(subject));
```
valkey@9.1.2:src/hashtable.h:124 `size_t hashtableSize(const hashtable *ht);`
valkey@9.1.2:src/intset.h:37 `uint32_t length;`
**Fix:** "A Set has no fixed member limit on 64-bit builds. Its size is bounded by available memory and `maxmemory`." Confirm with a maintainer before publishing; the same claim appears in topics/hashes.md:115 and topics/lists.md:396.

### F4 [minor] MISSING: Performance section omits the costly multi-set commands (`topics/sets.md:161`)
**Doc says:**
> Most set operations, including adding, removing, and checking whether an item is a set member, are O(1).
**Problem:** The only warning is about `SMEMBERS`. `SINTER`, `SUNION`, `SDIFF`, and their `STORE` variants are O(N) or O(N*M) and block the server on large sets just as `SMEMBERS` does. `SADD`/`SREM` are O(1) per member, not per call.
**Evidence:** valkey@9.1.2:src/commands/*.json
```
sinter: "O(N*M) worst case where N is the cardinality of the smallest set and M is the number of sets."
sunion: "O(N) where N is the total number of elements in all given sets."
sdiff:  "O(N) where N is the total number of elements in all given sets."
sadd:   "O(1) for each element added, so O(N) to add N elements when the command is called with multiple arguments."
```
**Fix:** Replace lines 161-165 with: "`SADD`, `SREM`, and `SISMEMBER` are O(1) per member. `SMEMBERS` is O(N) and `SINTER`, `SUNION`, and `SDIFF` are O(N) to O(N*M) in the size of the input sets. On sets with hundreds of thousands of members these block the server; use `SSCAN` to iterate, and `SINTERCARD` with `LIMIT` when you only need a count."

### F5 [minor] MISSING: No mention of set encodings or their configs (`topics/sets.md:155`)
**Doc says:** (nothing; Limits and Performance cover only size and complexity)
**Problem:** Small sets are stored compactly as `intset` or `listpack` and convert to `hashtable` past thresholds. This drives memory use and is the Valkey-specific detail a reader tuning sets needs. It is only listed bare in topics/memory-optimization.md:21-23.
**Evidence:** 9.1.2 live run:
```
> OBJECT ENCODING bikes:racing:france
listpack
> SADD s 1 2 3
> OBJECT ENCODING s
intset
> CONFIG GET set-max-*
set-max-intset-entries 512
set-max-listpack-entries 128
set-max-listpack-value 64
```
**Fix:** Add under Limits or a new "## How Valkey stores sets" heading: "Sets of integers with at most `set-max-intset-entries` (default 512) members use the compact `intset` encoding. Other sets with at most `set-max-listpack-entries` (default 128) members, each at most `set-max-listpack-value` (default 64) bytes, use `listpack`. Larger sets use a hashtable. See [Memory optimization](memory-optimization.md)."

### F6 [nit] STYLE: Fenced code blocks have no language tag (`topics/sets.md:28`)
**Doc says:**
> ```
> 127.0.0.1:6379> SADD bikes:racing:france bike:1
**Problem:** All nine blocks (lines 28, 40, 48, 54, 65, 80, 92, 108, 140) are untagged.
**Evidence:** See lines listed.
**Fix:** Use ```` ```text ```` (or the tag other topics pages use for valkey-cli sessions) on each block.

### F7 [nit] EXAMPLE: SMEMBERS output drops the quotes valkey-cli prints (`topics/sets.md:69`)
**Doc says:**
> 1) bike:3
> 2) bike:1
> 3) bike:2
**Problem:** Every other block shows quoted strings (`1) "bike:1"`). valkey-cli in interactive mode quotes bulk strings.
**Evidence:** Lines 50, 96, 118 use `"bike:N"`.
**Fix:** `1) "bike:1"` / `2) "bike:2"` / `3) "bike:3"`.

### F8 [nit] STYLE: First-person voice and weak front matter description (`topics/sets.md:74`)
**Doc says:**
> Here I've added three elements to my set and told Valkey to return all the elements.

and line 4: `Introduction to Sets`
**Problem:** First person is inconsistent with the rest of the page ("we", imperative). The description repeats the title and says nothing.
**Fix:** Line 74: "This adds three members and returns them all." Description: "Valkey Sets: unordered collections of unique strings, with membership tests, intersections, unions, and differences."

### F9 [nit] AI-NATIVE: "Examples" and "Tutorial" sections cover the same commands with no distinct purpose (`topics/sets.md:24`)
**Doc says:**
> ## Examples
> ...
> ## Tutorial
**Problem:** Both sections run `SADD`, `SISMEMBER`, and `SINTER` on the same keys. The headings say nothing about what each section answers, and the overlap causes F1.
**Fix:** Merge into one section headed "Working with sets: membership, intersection, union, and difference", keeping the Tutorial blocks and dropping lines 24-57 except the duplicate-member `SADD` demonstration.
