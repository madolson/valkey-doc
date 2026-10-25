VERDICT: NEEDS-WORK
COUNTS: blocker=1 minor=4 nit=10

# Lua API reference (`lua-api`)

https://valkey.io/topics/lua-api/

The reference is mostly accurate for libraries and reply helpers, but it documents `server.SERVER_VERSION` and `server.SERVER_VERSION_NUM`, which have never existed in any Valkey release; the real fields are `server.VALKEY_VERSION` and `server.VALKEY_VERSION_NUM`. Fix that first. The live page also carries the "under review" banner (Tracked: valkey-io/valkey-doc#302).

## Findings

### F1 [blocker] INACCURATE: `server.SERVER_VERSION` and `server.SERVER_VERSION_NUM` do not exist (`topics/lua-api.md:470`)
**Doc says:**
> ### <a name="server.server_version"></a> `server.SERVER_VERSION`
> * Since version: 7.2.5
> Returns the current Valkey server version as a Lua string.

and at line 496:
> ### <a name="server.redis_version_num"></a> `server.SERVER_VERSION_NUM`

**Problem:** Both fields are nil on 9.1.2. Valkey has exposed `VALKEY_VERSION`, `VALKEY_VERSION_NUM`, and `SERVER_NAME` since 7.2.5. A script that does `server.SERVER_VERSION .. ''` fails, and a script that compares `server.SERVER_VERSION_NUM` gets nil. `SERVER_NAME` is not documented anywhere on the page. topics/migration.md:42 has the same wrong names.
**Evidence:** valkey@9.1.2:src/modules/lua/script_lua.c:1706-1716:
```c
    lua_pushstring(lua, "VALKEY_VERSION_NUM");
    lua_pushnumber(lua, ctx->valkey_version_num);
    ...
    lua_pushstring(lua, "VALKEY_VERSION");
    ...
    lua_pushstring(lua, "SERVER_NAME");
```
The same names are in 7.2.5:src/script_lua.c:1387-1395, 8.0.0, 8.1.0, 9.0.0, and origin/unstable. Live 9.1.2:
```
SERVER_VERSION:      (nil)
SERVER_VERSION_NUM:  (nil)
VALKEY_VERSION:      "9.1.2"
VALKEY_VERSION_NUM:  (integer) 590082
SERVER_NAME:         "valkey"
REDIS_VERSION:       "7.2.4"
```
**Fix:** Rename the headings and anchors to `server.VALKEY_VERSION` (anchor `server.valkey_version`) and `server.VALKEY_VERSION_NUM` (anchor `server.valkey_version_num`). Add a `server.SERVER_NAME` entry: "Since version: 7.2.5. Returns the server name as a Lua string (`valkey`)." Under `REDIS_VERSION`, add "Valkey always reports `7.2.4`." Fix topics/migration.md:42 the same way.

### F2 [minor] INACCURATE: How a returned Lua boolean is encoded depends on `server.setresp()`, not on the client's `HELLO 3` (`topics/lua-api.md:629`)
**Doc says:**
> Regardless of the script's choice of protocol version set for replies with the [`server.setresp()` function] when it calls `server.call()` or `server.pcall()`, the user may opt-in to using RESP3 (with the `HELLO 3` command) ...
> * Lua Boolean -> [RESP3 Boolean reply](protocol.md#booleans)

**Problem:** The script's `setresp` version controls how a returned boolean is encoded. A RESP3 client gets a RESP3 boolean only if the script also called `server.setresp(3)`. Without that call, `true` comes back as `:1` and `false` as a null. Also, with a RESP2 client plus `setresp(3)`, `false` returns `:0`, not null as line 561 implies. Maps, sets, and doubles do follow the client protocol.
**Evidence:** valkey@9.1.2:src/modules/lua/script_lua.c:2162 `luaReplyToServerReply(ctx, call_ctx.resp, lua);` (`call_ctx.resp` is the script's setresp value). Lines 371-381 branch on that `resp_version` for `LUA_TBOOLEAN`. Raw socket against 9.1.2:
```
HELLO 3; EVAL "return true" 0                     -> b':1\r\n'
HELLO 3; EVAL "return false" 0                    -> b'_\r\n'
HELLO 3; EVAL "server.setresp(3); return true" 0  -> b'#t\r\n'
HELLO 2; EVAL "server.setresp(3); return false" 0 -> b':0\r\n'
HELLO 3; EVAL "return {map={a=1}}" 0              -> b'%1\r\n$1\r\na\r\n:1\r\n'
```
**Fix:** Replace the Boolean bullet with: "Lua Boolean -> [RESP3 Boolean reply](protocol.md#booleans), only when the script has called `server.setresp(3)`. Otherwise booleans follow the RESP2 rules (`true` -> integer 1, `false` -> null), even on a RESP3 connection." Remove "Regardless of the script's choice of protocol version ..." from the lead sentence.

### F3 [minor] INACCURATE: `server.set_repl()` is available in functions (`topics/lua-api.md:299`)
**Doc says:**
> ### `server.set_repl(x)`
> * Available in scripts: yes
> * Available in functions: no

**Problem:** `set_repl` and the `REPL_*` constants are registered in the API shared by scripts and functions, and they work inside `FCALL`.
**Evidence:** valkey@9.1.2:src/modules/lua/script_lua.c:1752-1755 (comment on `luaRegisterServerAPI`): "This function only handles fields common between Functions and LUA scripting." Lines 1810-1836 register `set_repl` and `REPL_*` there. Live 9.1.2:
```
FUNCTION LOAD "...server.register_function('w', function(keys) server.set_repl(server.REPL_NONE); server.call('set', keys[1], 'x'); return server.REPL_ALL end)"
FCALL w 1 k1
(integer) 3
```
In the same function, `type(server.replicate_commands)`, `type(server.breakpoint)`, and `type(server.debug)` returned `nil`, so those "no" entries are correct.
**Fix:** Change line 299 to `* Available in functions: yes`. Change line 314 to "when a script or function begins its execution".

### F4 [minor] INACCURATE: The default flag behavior list is wrong for Eval scripts that have no shebang (`topics/lua-api.md:414`)
**Doc says:**
> By default, Valkey assumes that all scripts read and write data.
> This results in the following behavior:
> ...
> 1. They can run in cluster mode, and are not able to run commands accessing keys of different hash slots.

**Problem:** An Eval script with no `#!` line can access keys in different slots. Only functions and shebang scripts get these defaults. The note at line 412 only points elsewhere, and eval-intro.md:290 points back here for "the defaults", so neither page states the difference.
**Evidence:** 9.1.2 cluster node (all slots on one node):
```
EVAL "server.call('set','a','1'); server.call('set','b','1'); return 1" 0
(integer) 1
EVAL "#!lua\nserver.call('set','a','1'); server.call('set','b','1'); return 1" 0
(error) ERR Script attempted to access keys that do not hash to the same slot script: on @user_script:2.
```
**Fix:** Change line 415 to "By default, Valkey assumes that functions and Eval scripts with a shebang (`#!`) read and write data." Then add after the list: "Eval scripts without a shebang keep the legacy behavior: they can access keys from different slots. They are still denied on stale replicas and under OOM only when they call a write or deny-oom command." Change line 412 to say that.

### F5 [minor] MISSING: The page never says which Lua builtins are available, and does not mention that `getfenv`, `setfenv`, and `newproxy` are disabled (`topics/lua-api.md:650`)
**Doc says:**
> The Valkey Lua runtime context always comes with several pre-imported libraries.

**Problem:** The page lists libraries but not base functions. Since 8.0.6, 8.1.4, and 9.0.0, `getfenv`, `setfenv`, and `newproxy` are removed unless the hidden, protected config `lua-enable-insecure-api` is set. Scripts that relied on them fail with "nonexistent global variable" and nothing in valkey-doc explains why (grep for `insecure-api`, `getfenv`, `newproxy` in topics/ and commands/ returns nothing). `print`, `dofile`, `loadfile`, `require`, and `debug` are also absent.
**Evidence:** valkey@9.1.2:src/modules/lua/script_lua.c:101-143 (`lua_builtins_allow_list`, `lua_builtins_deprecated` = `newproxy`, `setfenv`, `getfenv`), lines 1588-1597 (deprecated allowed only if `lua_enable_insecure_api`), lines 163-168 (`deny_list` = `dofile`, `loadfile`, `print`). valkey@9.1.2:src/config.c:3319 `createBoolConfig("lua-enable-insecure-api", "lua-enable-deprecated-api", MODIFIABLE_CONFIG | HIDDEN_CONFIG | PROTECTED_CONFIG, ..., 0, ...)`. Live:
```
EVAL "return type(getfenv)" 0
(error) ERR user_script:1: Script attempted to access nonexistent global variable 'getfenv' script: on @user_script:1.
CONFIG GET lua-enable-insecure-api -> "no"
```
**Fix:** Add a "Lua base functions" subsection under Runtime libraries: "Available: `assert`, `collectgarbage`, `coroutine`, `error`, `gcinfo`, `getmetatable`, `ipairs`, `load`, `loadstring`, `next`, `pairs`, `pcall`, `rawequal`, `rawget`, `rawset`, `select`, `setmetatable`, `tonumber`, `tostring`, `type`, `unpack`, `xpcall`, `_G`, `_VERSION`. Not available: `print`, `dofile`, `loadfile`, `require`, `debug`. `getfenv`, `setfenv`, and `newproxy` are disabled since 8.0.6, 8.1.4, and 9.0.0 for sandbox safety. They can be re-enabled with the protected config `lua-enable-insecure-api yes` (default `no`)."

### F6 [nit] EXAMPLE: The wrong-args error text does not match 9.1.2 (`topics/lua-api.md:136`)
**Doc says:**
> (error) ERR Wrong number of args calling Valkey command from script script: b0345693f4b77517a711221050e76d24ae60b7f7, on @user_script:1.

**Problem:** 9.1.2 does not print "Valkey" or the SHA.
**Evidence:**
```
(error) ERR Wrong number of args calling command from script script: on @user_script:1.
```
**Fix:** Replace it with the output above. Change the fence at line 134 from `lua` to `bash` or untagged, because the block is a CLI session.

### F7 [nit] EXAMPLE: The `server.log` output sample uses a pre-Redis-3 log format (`topics/lua-api.md:276`)
**Doc says:**
> [32343] 22 Mar 15:21:39 # Something is terribly wrong

**Problem:** The current default log format includes the role, the year, milliseconds, and a `<lua>` prefix.
**Evidence:** 9.1.2 log line:
```
17890:M 24 Oct 2026 00:47:58.640 # <lua> Something is terribly wrong
```
**Fix:** Replace the sample with the line above.

### F8 [nit] OUTDATED: "Until version: 7.0.0" suggests `server.replicate_commands()` was removed, but it still exists (`topics/lua-api.md:332`)
**Doc says:**
> * Until version: 7.0.0

**Problem:** In Eval scripts the function is still defined and returns 1 as a no-op. Readers may delete working compatibility calls or expect an error.
**Evidence:** Live 9.1.2: `EVAL "return type(server.replicate_commands)" 0` -> `function`. `EVAL "return server.replicate_commands()" 0` -> `(integer) 1`.
**Fix:** Replace "Until version: 7.0.0" with "Deprecated: since Redis OSS 7.0 it is a no-op that returns `true`. Kept for backward compatibility."

### F9 [nit] BROKEN: A duplicate anchor and an unlinked reference (`topics/lua-api.md:496`)
**Doc says:**
> line 496: `### <a name="server.redis_version_num"></a> `server.SERVER_VERSION_NUM``
> line 509: `### <a name="server.redis_version_num"></a> `server.REDIS_VERSION_NUM``
> line 629: `with the [`server.setresp()` function] when it calls`

**Problem:** Two headings share the anchor `server.redis_version_num`, so links to it land on the wrong field. On line 629, `[...]` has no target and renders as literal brackets.
**Fix:** Use a distinct anchor at line 496 (see F1). Line 629: `[`server.setresp()`](#server.setresp)`.

### F10 [nit] INACCURATE: The `double` conversion rule says "associative Lua table" (`topics/lua-api.md:635`)
**Doc says:**
> Lua table with a single `double` field to an associative Lua table -> [RESP3 double reply]

**Problem:** The `double` field must be a Lua number. valkey@9.1.2:src/modules/lua/script_lua.c:419-426 checks `t == LUA_TNUMBER`.
**Fix:** "Lua table with a single `double` field set to a Lua number -> RESP3 double reply."

### F11 [nit] STYLE: Typos and wrong example output
**Doc says / Fix:**
- topics/lua-api.md:23: "a context needs to be maintain" -> "maintained".
- topics/lua-api.md:24: "Valkey' keyspace" -> "Valkey's keyspace".
- topics/lua-api.md:181: "`error1` and `error2` are identical" -> "`reply1` and `reply2`" (the code uses `reply1`/`reply2`).
- topics/lua-api.md:257: "can be on of these" -> "one of".
- topics/lua-api.md:306: "that only be one of" -> "that can only be one of".
- topics/lua-api.md:411: "which may negatively impact if misused" -> "misused flags can cause incorrect behavior".
- topics/lua-api.md:589: `1) "Hello World!"` -> `2) "Hello World!"` (9.1.2 prints `2)`).
- topics/lua-api.md:597: "cont nils" -> "contain nils".
- topics/lua-api.md:607: "_3.333_" -> "_3.3333_".
- topics/lua-api.md:664 vs 813: the list says "_bitop_ library" but the section and the Lua global are `bit`. Use "_bit_ library".
- topics/lua-api.md:848: the heading repeats `bit.bnot(x)`, which already has its own section at line 844. Drop it from the heading.

### F12 [nit] STYLE: A heading level is wrong (`topics/lua-api.md:552`)
**Doc says:**
> ## Lua to RESP2 type conversion

**Problem:** Its siblings ("RESP2 to Lua", "RESP3 to Lua", "Lua to RESP3") are `###` under "## Data type conversion", so this heading breaks the outline.
**Fix:** `### Lua to RESP2 type conversion`.

### F13 [nit] STYLE: CLI sample blocks have no language tag
**Doc says:** Untagged fences at topics/lua-api.md:168, 191, 224, 243, 275, 387, 404, 581, 599, 720, 732, 746, 765, 776, 795, 806, 828, 839, 855.
**Fix:** Tag the valkey-cli sessions `bash` (or the site's convention for CLI sessions) and the log line at 275 `text`.

### F14 [nit] STYLE: The front matter description is weak (`topics/lua-api.md:4`)
**Doc says:**
> Executing Lua in Valkey

**Fix:** "Reference for the Lua 5.1 sandbox in Valkey: the server.* API, script and function flags, RESP-to-Lua type conversion, and bundled libraries (cjson, cmsgpack, struct, bit, os)."

### F15 [nit] AI-NATIVE: The heading "Additional notes about scripting" does not say what the section covers (`topics/lua-api.md:641`)
**Doc says:**
> ## Additional notes about scripting
> ### Using `SELECT` inside scripts

**Problem:** The only content is one note about `SELECT`, and the wrapper heading tells the reader nothing.
**Fix:** Remove the H2 and promote the note to `## Using SELECT inside scripts`.

## Unverified
- topics/lua-api.md:61: "circumvent the sandbox isn't hard" using debugging functionality or metatables. On 9.1.2, `debug` is nil and `getmetatable(_G).__index = nil` fails with "Attempt to modify a readonly table". Whether other bypasses exist was not tested. The sentence may be outdated, but I could not prove it.
