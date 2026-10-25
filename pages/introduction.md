VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=3 nit=4

# Introduction (`introduction`)

https://valkey.io/topics/introduction/

Accurate and all links resolve, but it reads as an inherited Redis-era overview: it omits what Valkey adds (modules via Valkey Bundle, Functions, multithreaded I/O) and never mentions Redis compatibility. Fix the missing-capabilities gap first.

## Findings

### F1 [minor] MISSING: Feature overview omits modules, Functions, and multithreaded I/O (`topics/introduction.md:7`)
**Doc says:**
> Valkey has built-in [replication](replication.md), [Lua scripting](eval-intro.md), [LRU eviction](lru-cache.md), [transactions](transactions.md), and different levels of [on-disk persistence](persistence.md)
**Problem:** The overview lists only the original data types and features. A new reader does not learn that Valkey is extensible with modules (JSON, Bloom, Search, packaged as Valkey Bundle), that server-side scripting includes Functions, or that it supports multithreaded I/O. These are the main differentiators and each has its own page.
**Evidence:** valkey-doc has topics/modules-intro.md, topics/valkey-bundle.md ("comes preloaded with a collection of supported modules"), and topics/functions-intro.md, none linked from this page. valkey@9.1.2:README.md:11: "It supports a wide range of native structures and an extensible plugin system for adding new data structures and access patterns."
**Fix:** Add after line 7: "Valkey can be extended with [modules](modules-intro.md). [Valkey Bundle](valkey-bundle.md) packages Valkey with the JSON, Bloom, and Search modules." Change "[Lua scripting](eval-intro.md)" to "[Lua scripting](eval-intro.md) and [Functions](functions-intro.md)". Add multithreaded I/O to the feature list, linking the io-threads section of the config docs.

### F2 [minor] MISSING: No mention of Redis origin or compatibility (`topics/introduction.md:6`)
**Doc says:**
> Valkey is an open source (BSD licensed), in-memory __data structure store__ used as a database, cache, message broker, and streaming engine.
**Problem:** The introduction never says Valkey is a fork of Redis or that it is protocol and command compatible with Redis OSS 7.2. Many readers arrive from Redis and this is the first thing they need to know. The first paragraph also does not say who the page is for.
**Evidence:** valkey@9.1.2:README.md:4: "This project was forked from the open source Redis project right before the transition to their new source available licenses." topics/migration.md exists and is not linked.
**Fix:** Append to line 6: "Valkey is a fork of Redis OSS 7.2 and stays compatible with its protocol, commands, and data files. If you are moving from Redis, see [Migration](migration.md)."

### F3 [minor] DUPLICATE: "Valkey also includes" list repeats line 7 with different link targets (`topics/introduction.md:23`)
**Doc says:**
> * [Transactions](transactions.md)
> * [Lua scripting](../commands/eval.md)
> * [LRU eviction of keys](lru-cache.md)
> * [Automatic failover](sentinel.md)
**Problem:** Four of the six bullets (lines 25, 27, 29, 30) restate features already listed on line 7. Lua scripting links to eval-intro.md on line 7 and to commands/eval.md on line 27, so readers land on different pages for the same feature.
**Evidence:** Line 7 already links transactions.md, eval-intro.md, lru-cache.md, sentinel.md.
**Fix:** Reduce the list to the items not already covered: "Valkey also supports [Pub/Sub](pubsub.md) messaging and [keys with a limited time-to-live](../commands/expire.md)." Delete lines 25, 27, 29, 30.

### F4 [nit] INACCURATE: "ANSI C 11" is not the language standard Valkey builds with (`topics/introduction.md:34`)
**Doc says:**
> Valkey is written in **ANSI C 11** with Atomics
**Problem:** C11 is an ISO standard, not ANSI, and the build uses the GNU dialect. It falls back to C99 when the compiler lacks `<stdatomic.h>`.
**Evidence:** valkey@9.1.2:src/Makefile:55-61:
```
C11_ATOMIC := $(shell sh -c 'echo "$(NUMBER_SIGN_CHAR)include <stdatomic.h>" > foo.c; \
	$(CC) -std=gnu11 -c foo.c -o foo.o > /dev/null 2>&1; \
...
ifeq ($(C11_ATOMIC),yes)
	STD+=-std=gnu11
else
	STD+=-std=c99
```
**Fix:** "Valkey is written in **C11** (GNU dialect) with C11 atomics and a few GCC/Clang built-ins like `__builtin_clz()`."

### F5 [nit] OUTDATED: Platform support understates what CI tests (`topics/introduction.md:38`)
**Doc says:**
> Supported hardware includes x86-64 (AKA amd64), x86 (32-bit) and AArch64 (64-bit ARM).
> It is also known to work on IBM z/Architecture like s390x and builds for this system are available from the Fedora distro.
**Problem:** s390x is tested in Valkey's own daily CI, so "known to work" with a Fedora pointer undersells it. The README describes support by endianness and word size, not a fixed CPU list.
**Evidence:** valkey .github/workflows/daily.yml:19 default jobs include "arm,s390x"; daily.yml:237 `test-s390x:`; ci.yml:247 `build-32bit:`. valkey@9.1.2:README.md:16-17: "We support big endian and little endian architectures, and both 32 bit and 64 bit systems."
**Fix:** "Valkey supports 32-bit and 64-bit, big-endian and little-endian systems. CI tests x86-64, x86 (32-bit), AArch64, and s390x."

### F6 [nit] STYLE: "LRU eviction" omits LFU and other policies; "MacOS" casing (`topics/introduction.md:7`)
**Doc says:**
> [LRU eviction](lru-cache.md)
**Problem:** The linked page covers LRU, LFU, random, and TTL policies (topics/lru-cache.md:3 "Overview of Valkey key eviction policies (LRU, LFU, etc.)"). The same narrow label is on line 29. Lines 35 and 36 write "MacOS"; Apple's name is "macOS". Line 38 uses "AKA".
**Fix:** "[key eviction (LRU, LFU)](lru-cache.md)" on line 7. "macOS" on lines 35 and 36. "x86-64 (amd64)" on line 38.

### F7 [nit] AI-NATIVE: Front matter description is generic (`topics/introduction.md:3`)
**Doc says:**
> description: Learn about the Valkey open source project
**Problem:** The description names no capability, so it does not help a reader or search result distinguish this page.
**Fix:** `description: What Valkey is: an open source, in-memory data store with rich data types, replication, persistence, clustering, and modules`
