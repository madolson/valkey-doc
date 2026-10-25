VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=5

# Sorted Sets (`sorted-sets`)

https://valkey.io/topics/sorted-sets/

The page is a lightly edited Redis-era tutorial. It names the wrong data structure (B+ tree instead of skip list), shows one wrong example output, and still has leftover text from the old "hackers" example. Fix the implementation note first.

## Findings

### F1 [minor] INACCURATE: Sorted sets are a skip list plus hash table, not a B+ tree (`topics/sorted-sets.md:50`)
**Doc says:**
> Implementation note: Sorted sets are implemented via a dual-ported data structure containing both a B+ tree and a hash table, so every time we add an element Valkey performs an O(log(N)) operation. That's good, but when we ask for sorted elements Valkey does not have to do any work at all, it's already sorted.

**Problem:** Valkey has no B+ tree. Large sorted sets use a skip list plus a hash table, and small ones use a listpack. "Does not have to do any work at all" also contradicts the page's own Performance section (line 231), which says ZRANGE is O(log(n) + m).
**Evidence:** valkey@9.1.2:src/t_zset.c:44: `* At the same time the elements are added to a skip list mapping scores`. valkey@9.1.2:src/server.h:1491: `/* ZSETs use a specialized version of Skiplists */`. valkey@9.1.2:src/server.h:769: `#define OBJ_ENCODING_SKIPLIST 7`. A live 9.1.2 run on the page's example key gives `OBJECT ENCODING racer_scores` -> `listpack`, with `zset-max-listpack-entries 128` and `zset-max-listpack-value 64`.
**Fix:** "Implementation note: Small sorted sets (up to `zset-max-listpack-entries` members, default 128, each no longer than `zset-max-listpack-value` bytes, default 64) are stored as a compact listpack. Larger sets use a skip list plus a hash table. Adding a member is O(log(N)). Because members are kept in order, a range read costs O(log(N) + M) and needs no sorting. See [Memory optimization](memory-optimization.md)."

### F2 [minor] EXAMPLE: ZREVRANK output is wrong (`topics/sorted-sets.md:139`)
**Doc says:**
> 127.0.0.1:6379> ZREVRANK racer_scores "Norem"
> (integer) 3

**Problem:** At this point the set holds Norem(10), Royce(10), Prickett(14). In descending order Norem has rank 2.
**Evidence:** Ran the page's commands in order on 9.1.2:
```
ZRANGE racer_scores 0 -1   -> Norem, Royce, Prickett
ZRANK racer_scores Norem   -> (integer) 0
ZREVRANK racer_scores Norem -> (integer) 2
```
**Fix:** Change line 139 to `(integer) 2`.

### F3 [minor] CONFUSING: Text left over from the old "hackers" example contradicts the racer example (`topics/sorted-sets.md:44`, `:47`, `:153`)
**Doc says:**
> `ZADD` is also variadic, so you are free to specify multiple score-value pairs, even if this is not used in the example above.

> With sorted sets it is trivial to return a list of hackers sorted by their birth year because actually *they are already sorted*.

> For example, let's add again our list of famous hackers, but this time using a score of zero

**Problem:** Line 37 does use variadic ZADD (four pairs), and the example is about racers and points, not hackers and birth years. Readers have to stop and work out which data the prose means.
**Fix:** Line 44 to 45: "`ZADD` is variadic: the third call above adds four score-member pairs at once." Line 47 to 48: "Because the set is always kept in order, returning the racers sorted by score needs no extra work." Line 153: "For example, let's add our racers again, this time with a score of zero for every member."

### F4 [minor] OUTDATED: Teaches the legacy range commands and never mentions unified ZRANGE (`topics/sorted-sets.md:54`, `:97`, `:150`)
**Doc says:**
> Note that the `ZRANGE` order is low to high, while the `ZREVRANGE` order is high to low

> We use the `ZRANGEBYSCORE` command to do it

> The main commands to operate with lexicographical ranges are `ZRANGEBYLEX`, `ZREVRANGEBYLEX`, `ZREMRANGEBYLEX` and `ZLEXCOUNT`.

**Problem:** Since 6.2, `ZRANGE` covers all of these with `REV`, `BYSCORE`, `BYLEX`, and `LIMIT`. The command pages already point readers to it (commands/zrangebyscore.md:42: "## Alternative ... `ZRANGE` with the `BYSCORE` argument."). This page never mentions the options, so new users learn the older commands first.
**Evidence:** valkey@9.1.2:src/commands/zrange.json:11-12: `"6.2.0", "Added the \`REV\`, \`BYSCORE\`, \`BYLEX\` and \`LIMIT\` options."`
**Fix:** Use `ZRANGE racer_scores 0 -1 REV`, `ZRANGE racer_scores -inf 10 BYSCORE`, and `ZRANGE racer_scores [A [L BYLEX` in the examples, and add one sentence: "`ZREVRANGE`, `ZRANGEBYSCORE`, `ZREVRANGEBYSCORE`, `ZRANGEBYLEX`, and `ZREVRANGEBYLEX` still work, but `ZRANGE` with `REV`, `BYSCORE`, or `BYLEX` (since 6.2) replaces them." Change line 150 to list `ZRANGE ... BYLEX`, `ZREMRANGEBYLEX`, and `ZLEXCOUNT`.
Tracked: valkey-io/valkey-doc#304

### F5 [nit] STYLE: Heading hierarchy is broken and headings say little (`topics/sorted-sets.md:94`, `:142`, `:184`, `:199`)
**Doc says:**
> ### Operating on ranges
> ### Lexicographical scores
> Updating the score: leaderboards
> ---
> ## Examples

**Problem:** The first headings are H3 with no H2 above them. Line 184 is a setext H2 (`---`), unlike the ATX headings on the rest of the page. "Lexicographical scores" is wrong: the section is about lexicographical ranges over members that share a score. "Examples" does not say what the example shows.
**Fix:** Use `##` for every top-level section: `## Range queries by score`, `## Lexicographical ranges`, `## Updating scores for leaderboards`. Rename "Examples" to `## Example: ZADD vs ZINCRBY`, or merge it into the leaderboard section.

### F6 [nit] AI-NATIVE: Weak description, no opening statement of scope, and tutorial filler (`topics/sorted-sets.md:4`, `:187`, `:195`)
**Doc says:**
> description: > Introduction to Sorted Sets

> Just a final note about sorted sets before switching to the next topic.

> The typical application is a Facebook game

**Problem:** The description carries no information. Line 187 refers to a "next topic" that does not exist on this page. The Facebook reference is dated filler.
**Fix:** Description: "How Valkey sorted sets order members by score, with range, rank, and lexicographical queries, leaderboard examples, and performance characteristics." Delete line 187. Line 195: "The typical application is a game leaderboard that shows the top-N users and each user's rank."

### F7 [nit] MISSING: Score precision limits not stated or linked (`topics/sorted-sets.md:19`)
**Doc says:**
> every element in a sorted set is associated with a floating point value, called *the score*

**Problem:** Leaderboard and indexing users need to know that only integers within ±2^53 are exact. commands/zadd.md:28-31 explains this, but this page does not link to it.
**Fix:** Append: "Scores are IEEE 754 doubles, so integers are exact only between -(2^53) and 2^53. See [ZADD](../commands/zadd.md)."

### F8 [nit] STYLE: Typos (`topics/sorted-sets.md:11`, `:221`, `:230`)
**Doc says:**
> maintain  ordered lists

> assuming the sorted is in ascending order

> with large returns values

**Fix:** "maintain ordered lists", "assuming the sorted set is in ascending order", "with large return values".

### F9 [nit] STYLE: Fenced code blocks have no language tag (`topics/sorted-sets.md:32`, `:56`, `:78`, `:100`, `:115`, `:135`, `:156`, `:202`)
**Doc says:**
> ```

**Problem:** None of the eight CLI blocks has a language tag. The block at line 202 also sits directly under a bullet with no blank line, so it renders loosely attached to the list.
**Fix:** Use a consistent tag on every block (for example ```` ```text ```` or whatever the site uses for valkey-cli sessions), and put a blank line before the block at line 202, or turn the bullet at line 201 into a paragraph.
