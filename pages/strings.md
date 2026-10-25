VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=3 nit=5

# Strings (`strings`)

https://valkey.io/topics/strings/

The page is accurate where it's specific, but thin. It points readers to legacy commands (`GETSET`, `SETNX` for locks, `SUBSTR`) that their own command pages list as superseded. It also states the 512 MB limit as fixed without naming the config that controls it. Fix the legacy-command guidance first.

## Findings

### F1 [minor] OUTDATED: Page recommends superseded commands GETSET, SETNX-for-locks, SUBSTR (`topics/strings.md:43`)
**Doc says:**
> the `GETSET` command sets a key to a new value, returning the old value as the result. (line 43)

> `SETNX` stores a string value only if the key doesn't already exist. Useful for implementing locks. (line 103)

> be careful with the `SUBSTR`, `GETRANGE`, and `SETRANGE` commands (line 121)

**Problem:** The command pages for all three name a replacement. This page presents the legacy form as the main way to do the job. The `SETNX` page also steers people away from the `SETNX` locking pattern.
**Evidence:** commands/getset.md:23-25: "## Alternative / `SET` with the `GET` argument." commands/setnx.md:8: "`SET` with the `NX` argument." and setnx.md:25: "The following pattern is discouraged in favor of [the Redlock algorithm](../topics/distlock.md)". commands/substr.md:10-12: "## Alternative / `GETRANGE`". valkey@9.1.2:src/commands/set.json gives the `GET` argument `"since": "6.2.0"`.
**Fix:** Line 43-49: use `SET key 0 GET` for the counter-reset pattern, and mention `GETSET` only as the older equivalent. Line 103: "`SET key value NX EX <seconds>` stores a value only if the key doesn't exist, with an expiry. Use it for simple single-instance locks. See [distributed locks](distlock.md)." Line 121: drop `SUBSTR` (it's an alias of `GETRANGE`).

### F2 [minor] INACCURATE: 512 MB limit stated as absolute; it's the `proto-max-bulk-len` default (`topics/strings.md:29`)
**Doc says:**
> A value can't be bigger than 512 MB. (line 29)

> By default, a single String can be a maximum of 512 MB. (line 96)

**Problem:** Line 29 says the limit is absolute. Line 96 says "by default" but doesn't name the setting. The cap is `proto-max-bulk-len`, and it can be changed at runtime.
**Evidence:** valkey@9.1.2:src/config.c:3461: `createLongLongConfig("proto-max-bulk-len", NULL, DEBUG_CONFIG | MODIFIABLE_CONFIG, 1024 * 1024, LONG_MAX, server.proto_max_bulk_len, 512ll * 1024 * 1024, ...)`. valkey@9.1.2:src/t_string.c:51: `if (total > server.proto_max_bulk_len || ...`. Live run on 9.1.2:
```
$ valkey-cli -p 21460 config set proto-max-bulk-len 1gb
OK
$ valkey-cli -p 21460 setrange big 600000000 x
600000001
```
**Fix:** Line 29: drop the size sentence. Line 96: "A single string value is limited to 512 MB by default. The limit is set by `proto-max-bulk-len`, which also caps request bulk size. Values this large are slow to read, write, replicate, and evict, so keep values small." Protocol.md:196 already names the config this way.

### F3 [minor] MISSING: No mention of expiration on SET, although the page opens with caching (`topics/strings.md:100`)
**Doc says:**
> They're often used for caching (line 10)

**Problem:** "Getting and setting Strings" lists `SET`, `SETNX`, `GET`, `MGET`. It never mentions `SET ... EX/PX/EXAT/PXAT/KEEPTTL`, `GETEX`, or `GETDEL`. A cache entry with no TTL is the most common mistake for the use case the page leads with. `MSET` is in the example at line 56 but missing from the command list.
**Evidence:** valkey@9.1.2:src/commands/set.json defines `EX`/`PX` (`"since": "2.6.12"`), `KEEPTTL` (`"since": "6.0.0"`), `EXAT`/`PXAT`/`GET` (`"since": "6.2.0"`). commands/getex.md and commands/getdel.md exist.
**Fix:** Add to the list at line 102-105: "`SET key value EX <seconds>` (or `PX`, `EXAT`, `PXAT`, `KEEPTTL`) sets a value with an expiry, which is what you want for cache entries." Also add "`GETEX` reads a value and changes its expiry.", "`GETDEL` reads and deletes a value.", and "`MSET` sets multiple keys in one command."

### F4 [nit] INACCURATE: Performance complexity for SETRANGE is misstated (`topics/strings.md:121`)
**Doc says:**
> be careful with the `SUBSTR`, `GETRANGE`, and `SETRANGE` commands, which can be O(n).

**Problem:** "n" is undefined. For `GETRANGE` it's the length of the returned string. For `SETRANGE` it's the length of the value argument, plus any zero-padding allocation when the offset is past the end. `MGET`/`MSET` are O(N) in the number of keys and aren't mentioned.
**Evidence:** valkey@9.1.2:src/commands/getrange.json: `"O(N) where N is the length of the returned string..."`. setrange.json: `"O(1), not counting the time taken to copy the new string in place... Otherwise, complexity is O(M) with M being the length of the value argument."` mget.json: `"O(N) where N is the number of keys to retrieve."`
**Fix:** "Most string commands are O(1). `GETRANGE` is O(N) in the length returned. `SETRANGE` and `APPEND` are O(M) in the length written, and `SETRANGE` past the end of the string allocates and zero-fills up to the offset. `MGET` and `MSET` are O(N) in the number of keys."

### F5 [nit] AI-NATIVE: Conversational filler and first person (`topics/strings.md:12`)
**Doc says:**
> Since Valkey keys are strings, when we use the string type as a value too, we are mapping a string to another string. (line 12)

> As you can see using the `SET` and the `GET` commands are the way we set and retrieve a string value. (line 23)

> The `SET` command has interesting options, that are provided as additional arguments. For example, I may ask `SET` to fail if the key already exists (lines 31-32)

> When `MGET` is used, Valkey returns an array of values. (line 64)

> Even if strings are the basic values of Valkey, there are interesting operations you can perform with them. (line 67)

> Internally it's always the same command, acting in a slightly different way. (lines 82-83)

**Problem:** These lines carry no information beyond the examples, and "I may ask" is first person left over from the old tutorial. Lines 82-83 describe an implementation detail readers can't act on.
**Fix:** Cut lines 12-13, 64, and 82-83. Line 23: "`SET` stores a value and `GET` retrieves it." Lines 31-33: "`SET NX` succeeds only if the key doesn't exist. `SET XX` succeeds only if it does:". Line 67: "`INCR` increments a string holding an integer atomically:".

### F6 [nit] STYLE: Heading levels skip and the first heading is H3 (`topics/strings.md:66`)
**Doc says:**
> ### Strings as counters

**Problem:** The first heading is H3 with no H2 parent. The next one (line 94, `## Limits`) is H2, so the outline is wrong. The opening material at lines 7-64 has no heading.
**Fix:** Change line 66 to `## Strings as counters`. Optionally add `## Setting and getting values` before line 16.

### F7 [nit] EXAMPLE: Fenced blocks have no language tag (`topics/strings.md:16`)
**Doc says:**
> ```` ``` ```` (lines 16, 35, 55, 70)

**Problem:** None of the four example blocks has a language tag. Their output matched 9.1.2 when run live.
**Fix:** Use ```` ```text ```` (or the tag other topic pages use for valkey-cli sessions).

### F8 [nit] AI-NATIVE: Front matter description is weak (`topics/strings.md:4`)
**Doc says:**
> Introduction to Strings

**Problem:** It repeats the title and doesn't say what the page covers.
**Fix:** `description: "Valkey string type: storing text and binary values, atomic counters, size limits, and performance of string commands."`
