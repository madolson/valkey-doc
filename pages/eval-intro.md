VERDICT: NEEDS-WORK
COUNTS: blocker=1 minor=4 nit=3

# Scripting with Lua (`eval-intro`)

https://valkey.io/topics/eval-intro/

The examples all run on 9.1.2. The script cache sections are wrong, though. They say cached scripts stay until restart or `SCRIPT FLUSH`, but since Valkey 8.0 scripts cached by `EVAL` are LRU-evicted once there are 500 of them. Fix that first.

## Findings

### F1 [blocker] INACCURATE: Script cache sections omit EVAL script LRU eviction (`topics/eval-intro.md:188`)
**Doc says:**
> During normal operation, an application's scripts are meant to stay indefinitely in the cache (that is, until the server is restarted or the cache being flushed).

> The only way to flush the script cache is by explicitly calling the `SCRIPT FLUSH` command.

> Practically speaking, it is much simpler for the client to assume that in the context of a given connection, cached scripts are guaranteed to be there unless the administrator explicitly invoked the `SCRIPT FLUSH` command.

Related: line 136 ("Every script you execute with `EVAL` is stored in a dedicated cache"), line 162 (lists restart, failover, and `SCRIPT FLUSH` as the only ways scripts disappear), line 215 (`SCRIPT EXISTS` returns 0 only if the script was never loaded or was flushed).
**Problem:** Since 8.0, scripts cached by `EVAL`/`EVAL_RO` are LRU-evicted once the cache reaches 500 of them. Scripts added with `SCRIPT LOAD` are never evicted. A client that runs `EVAL` once and then pipelines `EVALSHA`, trusting line 202, will get `NOSCRIPT` on a running server that nobody flushed.
**Evidence:** `valkey@9.1.2:src/eval.c:339-359`:
```
 * ... we implement script eviction
 * only for these (not for one loaded with SCRIPT LOAD).
#define LRU_LIST_LENGTH 500
    while (listLength(evalCtx.scripts_lru_list) >= LRU_LIST_LENGTH) {
        ...
        evalDeleteScript(c, oldest);
        server.stat_evictedscripts++;
```
Live 9.1.2: ran `EVAL "return 'first'" 0`, then `SCRIPT LOAD "return 'loaded'"`, then 500 distinct `EVAL`s:
```
EVALSHA <sha1 of first> 0   -> NOSCRIPT No matching script.
EVALSHA <sha1 of loaded> 0  -> loaded
evicted_scripts:1
```
commands/eval.md:18 already documents this ("Starting from Valkey 8.0, scripts loaded with `EVAL` ... will be deleted ... (least recently used order)").
**Fix:** Add to "Script cache semantics": "Since Valkey 8.0, scripts cached by `EVAL` or `EVAL_RO` are evicted in least recently used order once 500 such scripts are cached. Scripts added with `SCRIPT LOAD` are never evicted. The `evicted_scripts` field in `INFO stats` counts evictions." At line 188, replace "until the server is restarted or the cache being flushed" with "until the server restarts, the cache is flushed, or (for scripts cached by `EVAL`) the script is evicted". At line 192, change "The only way to flush" to "The only way to explicitly flush". At line 202, change the guarantee to cover only scripts added with `SCRIPT LOAD`. At line 215, add "or was evicted" to the meaning of 0.

### F2 [minor] OUTDATED: "single scripting engine" is outdated in 9.x (`topics/eval-intro.md:26`)
**Doc says:**
> Presently, Valkey supports a single scripting engine, the [Lua 5.1](https://www.lua.org/) interpreter.
**Problem:** Valkey 9.0 added a module API for registering scripting engines, and 9.1 moved Lua itself into a module. The `#!` shebang names an engine, so `#!<engine>` selects any registered engine. Lua is the only built-in engine, not the only possible one.
**Evidence:** `valkey@9.1.2:src/valkeymodule.h:2304`: `VALKEYMODULE_API int (*ValkeyModule_RegisterScriptingEngine)(...)`. `valkey@9.1.2:src/eval.c:254`: `*err = sdsnew("Invalid engine in script shebang");`, and `eval.c:296`: `// When no shebang is declared, assume the engine is LUA.` `9.1.2:00-RELEASENOTES:161`: "Move Lua scripting engine into a Valkey module (#2858)".
**Fix:** "Valkey ships with one built-in scripting engine, the [Lua 5.1](https://www.lua.org/) interpreter. Since Valkey 9.0, modules can register additional engines, which a script selects with its `#!<engine>` shebang line." At line 268, say that `lua` in `#!lua` is the engine name.

### F3 [minor] INACCURATE: NOSCRIPT error text is missing the trailing period (`topics/eval-intro.md:171`)
**Doc says:**
```
(error) NOSCRIPT No matching script
```
**Problem:** The real error text ends with a period. Readers who match the string exactly will copy the wrong text.
**Evidence:** `valkey@9.1.2:src/server.c:2120`: `shared.noscripterr = createSharedString("-NOSCRIPT No matching script.\r\n");`. Live: `EVALSHA ffff...ffff 0` returns `NOSCRIPT No matching script.`
**Fix:** `(error) NOSCRIPT No matching script.`

### F4 [minor] OUTDATED: Version notes cite "7.0" as if it were a Valkey release (`topics/eval-intro.md:31`)
**Doc says:**
> As of version 7.0, [Valkey Functions](functions-intro.md) offer an alternative approach

> but this was removed in 7.0. (line 235)
**Problem:** Valkey has no 7.0 release (its first was 7.2.4, forked from Redis OSS 7.2). Line 266 correctly says "Redis OSS 7.0", so the page is inconsistent, and a bare "7.0" suggests a Valkey version gate that does not exist. In every Valkey release, Functions exist, verbatim replication is gone, and flags are supported.
**Evidence:** Line 266: "starting with Redis OSS 7.0, there's a way to declare flags". topics/lua-api.md:303 uses "Since Redis OSS 7.0 (and Valkey)".
**Fix:** Line 31: "[Valkey Functions](functions-intro.md) offer an alternative approach...". Delete "As of version 7.0". Line 235: "...but this was removed in Redis OSS 7.0 and is not available in Valkey." Line 266: "Every Valkey version lets you declare flags...".

### F5 [minor] CONFUSING: Pipelining advice is garbled (`topics/eval-intro.md:184`)
**Doc says:**
> Therefore, a client library's implementation should revert to using plain `EVAL` of parameterized in the context of a pipeline.
**Problem:** The sentence is missing words. It also skips the alternative that line 218 recommends: call `SCRIPT LOAD` before the pipeline. That choice is now the safer one because of F1.
**Fix:** "Therefore, inside a pipeline a client should either use plain `EVAL` with a parameterized script, or call `SCRIPT LOAD` before sending the pipeline so that `EVALSHA` finds the script."

### F6 [nit] STYLE: Typos and broken grammar (`topics/eval-intro.md:88`)
**Doc says:**
> both _Hello_ and _Parameterization!_ regular input arguments (88)

> as can been seen above (105)

> Most of [Valkey' clients](../clients/) (175)

> ensure that `EVALSHA` doesn't not fail (218)

> as is the case with  (for example, `DEL` and `LREM`) (250)

> Valkey' memory usage (252)

> pass any arguments needed for to execute them (65)
**Fix:** "are regular input arguments", "as can be seen", "Valkey clients", "doesn't fail", "as is the case with `DEL` and `LREM`", "Valkey's memory usage", "needed to execute them".

### F7 [nit] CONFUSING: "entirely different, but ... perfectly identical" (`topics/eval-intro.md:55`)
**Doc says:**
> the application could send these two entirely different, but at the same time perfectly identical scripts:
**Problem:** The sentence contradicts itself, so the reader has to work out that the scripts have the same shape but different literals.
**Fix:** "the application could send these two scripts, which differ only in a literal value:"

### F8 [nit] STYLE: Fenced blocks lack language tags, and the front matter description is weak (`topics/eval-intro.md:39`)
**Doc says:** The ` ``` ` blocks at lines 39, 57, 69, 95, 120, 151, 169, and 270 have no language tag. The front matter description is "Executing Lua in Valkey".
**Fix:** Tag the CLI sessions as `bash` (or `text`) and the line 270 block as `lua`. Change the description to: "Run Lua scripts atomically on the server with EVAL and EVALSHA, manage the script cache, and declare script flags."
