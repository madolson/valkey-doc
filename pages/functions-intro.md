VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=5 nit=5

# Functions (`functions-intro`)

https://valkey.io/topics/functions-intro/

The tutorial examples mostly run as written on 9.1.2, but the "Function flags" section shows the wrong error text and lists failure cases that return different errors. Fix that section first.

## Findings

### F1 [minor] EXAMPLE: FCALL_RO error text is wrong, and the replica scenario returns a different error (`topics/functions-intro.md:388`)
**Doc says:**
> We can try executing them using `FCALL_RO` against a read-only replica.
> ```
> 127.0.0.1:6379> FCALL_RO my_hgetall 1 myhash
> (error) ERR Can not execute a function with write flag using fcall_ro.
> ```
**Problem:** The 9.1.2 error string is different ("a script", "*_ro command"). Also, the example says it runs against a read-only replica, but on a replica 9.1.2 returns READONLY, not the `*_ro` error. The `*_ro` error only shows up on a primary.
**Evidence:** valkey@9.1.2:src/script.c:184:
```
addReplyError(caller, "Can not execute a script with write flag using *_ro command.");
```
Live 9.1.2. Port 20420 is the primary and 20421 is its replica. `mylib` is loaded with no flags:
```
$ valkey-cli -p 20420 FCALL_RO my_hgetall 1 myhash
ERR Can not execute a script with write flag using *_ro command.
$ valkey-cli -p 20421 FCALL_RO my_hgetall 1 myhash
READONLY You can't write against a read only replica.
```
**Fix:** Change line 388 to "We can try executing them using `FCALL_RO`:". Change line 392 to `(error) ERR Can not execute a script with write flag using *_ro command.`

### F2 [minor] INACCURATE: "this error" is not returned in two of the three listed cases (`topics/functions-intro.md:397`)
**Doc says:**
> The server will reply with this error in the following cases:
> 1. Executing a function with `FCALL` against a read-only replica.
> 2. Using `FCALL_RO` to execute a function.
> 3. A disk error was detected (Valkey is unable to persist so it rejects writes).
**Problem:** Only case 2 produces the `*_ro` error. Case 1 returns READONLY. Case 3 returns MISCONF. The same list, worded correctly, is already in topics/lua-api.md:420-430 (`no-writes` under Script flags).
**Evidence:** valkey@9.1.2:src/script.c:154-184. The checks run in order: READONLY for a replica, then MISCONF for a disk error, then `*_ro`:
```
addReplyError(caller, "-READONLY Can not run script with write flag on readonly replica");
...
"-MISCONF %s is configured to save RDB snapshots, " ... "Writable scripts are blocked. Use 'no-writes' flag for read only scripts."
...
if (ro) { addReplyError(caller, "Can not execute a script with write flag using *_ro command.");
```
Live 9.1.2, on the replica: `FCALL my_hgetall 1 myhash` returned `READONLY You can't write against a read only replica.`
**Fix:** Replace lines 397-401 with: "Without the `no-writes` flag, the server also rejects the function on a read-only replica (`READONLY` error) and when a disk error stops it from persisting (`MISCONF` error)." Keep the flag behavior in lua-api.md#script_flags as the canonical description. This page links to it already.

### F3 [minor] INACCURATE: functions are persisted in RDB as well as AOF (`topics/functions-intro.md:45`)
**Doc says:**
> Functions are also persisted to the AOF file and replicated from primary to replicas
**Problem:** Readers who run RDB only will conclude that their functions are not persisted. RDB snapshots include function libraries.
**Evidence:** valkey@9.1.2:src/rdb.h:147:
```
#define RDB_OPCODE_FUNCTION2 245       /* function library data */
```
**Fix:** "Functions are persisted in RDB snapshots and the AOF, and replicated from primary to replicas, so they are as durable as the data itself."

### F4 [minor] OUTDATED: the engine is not "always lua" (`topics/functions-intro.md:72`)
**Doc says:**
> like the language (always "lua") and the library name.
**Problem:** Since 8.1, modules can register scripting engines, and the shebang names the engine. In 9.x, Lua itself is a scripting engine built as a module. The page says "Functions are written in Lua 5.1" (line 60) and never mentions this.
**Evidence:** valkey@9.1.2:00-RELEASENOTES:161 "Move Lua scripting engine into a Valkey module (#2858)". 8.1.0 release notes, line 150: "Adds support for scripting engines as Valkey modules (#1277, #1497)". valkey@9.1.2:src/valkeymodule.h:2304 `ValkeyModule_RegisterScriptingEngine`.
**Fix:** "like the engine name (`lua` for the built-in Lua 5.1 engine, or the name of an engine registered by a module, see [Scripting Engine API](modules-api-ref.md#section-scripting-engine-api)) and the library name."

### F5 [minor] MISSING: no pointer to the busy-script limit or FUNCTION KILL (`topics/functions-intro.md:58`)
**Doc says:**
> Because running a function blocks the Valkey server, functions are meant to finish executing quickly, so you should avoid using long-running functions.
**Problem:** The page never says what happens when a function runs too long: after `busy-reply-threshold` (default 5000 ms) the server returns BUSY to other clients, and `FUNCTION KILL` stops a function that has not written anything. That is the operational failure mode readers need. topics/programmability.md:106-116 covers it, but this page does not link there.
**Evidence:** valkey@9.1.2:src/config.c:3454:
```
createLongLongConfig("busy-reply-threshold", "lua-time-limit", MODIFIABLE_CONFIG, 0, LONG_MAX, server.busy_reply_threshold, 5000, ...
```
**Fix:** Append: "If a function runs longer than `busy-reply-threshold` (default 5000 ms), the server replies BUSY to other clients until the function finishes or is stopped with [`FUNCTION KILL`](../commands/function-kill.md). See [Maximum execution time](programmability.md#maximum-execution-time)."

### F6 [nit] EXAMPLE: log lines lack the `<lua>` prefix that 9.x prints (`topics/functions-intro.md:356`)
**Doc says:**
> 20075:M 1 Jan 2022 16:53:57.688 # Hash key name not provided
**Evidence:** Live 9.1.2 log:
```
12832:M 24 Oct 2026 00:45:54.232 # <lua> Hash key name not provided
12832:M 24 Oct 2026 00:45:54.234 # <lua> Only one key name is allowed
```
The prefix comes from module logging, valkey@9.1.2:src/module.c:8180 `"<%s> "`, because Lua is now a module engine.
**Fix:** Add `<lua> ` after `# ` on lines 356 and 357.

### F7 [nit] STYLE: fenced blocks with no language tag (`topics/functions-intro.md:75`)
**Doc says:** Bare ```` ``` ```` fences at lines 75, 81, 106, 167, 235, 248, 345, 354, 390, 424.
**Problem:** These fences have no language tag.
**Fix:** Tag the CLI sessions (81, 106, 167, 235, 248, 345, 390, 424) as `text` or `bash`. Tag the shebang (75) and the log (354) as `text`.

### F8 [nit] STYLE: grammar errors and a dead variable in the example (`topics/functions-intro.md:8`)
**Doc says:**
> This feature is as a complement to [EVAL scripts]
> The core use cases for `EVAL` scripts is executing (line 14)
> this name can later be used `FUNCTION LIST` and `FUNCTION DELETE` (line 113)
> For the complete documentation flags, please refer to (line 434)
> `local hash = keys[1]` / `return server.call('HGET', keys[1], ...)` (lines 334-335)
**Fix:** "This feature complements EVAL scripts." / "The core use case for `EVAL` scripts is" / "can later be used with `FUNCTION LIST` and `FUNCTION DELETE`" / "For complete documentation of flags, see" / at line 335 use `hash` instead of `keys[1]`.

### F9 [nit] OUTDATED: "introduced in version 7.0" has no product name (`topics/functions-intro.md:34`)
**Doc says:**
> functions were introduced in version 7.0.
**Problem:** Valkey has no 7.0 release. Its first release was 7.2.
**Fix:** "functions were introduced in Redis OSS 7.0 and are available in all Valkey releases."

### F10 [nit] AI-NATIVE: weak description and opening, and a rhetorical heading (`topics/functions-intro.md:4`)
**Doc says:**
> description: Scripting with functions stored on the server
> ## What's wrong with EVAL?
**Problem:** The first paragraph does not say what the page covers or who it is for. The first heading is a rhetorical question, and the section's own first sentence answers "nothing".
**Fix:** Change the description to "Store Lua libraries on the server with FUNCTION LOAD and call them with FCALL: persistence, replication, cluster loading, and flags." Rename the heading to "Functions compared with EVAL scripts". Rewrite lines 7-8 as: "Valkey Functions let you load libraries of server-side code once with `FUNCTION LOAD` and call them by name with `FCALL`. Valkey persists and replicates them like data. This page is for developers moving from `EVAL` scripts or writing server-side logic for the first time."
