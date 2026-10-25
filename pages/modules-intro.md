VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=10 nit=6

# Modules Introduction (`modules-intro`)

https://valkey.io/topics/modules-intro/

The page is a lightly renamed Redis-era guide. Several API claims are wrong for 9.1.2 when tested against a real module: `ValkeyModule_Call` errno values, `Call`/`Replicate` ordering, and lists described as complete that are not. The most important fix is the `MODULE LOAD` instructions. They fail on a default 9.1.2 server because `enable-module-command` defaults to `no`, and no valkey-doc page documents that setting.

## Findings

### F1 [minor] MISSING: `MODULE LOAD` fails by default, `enable-module-command` never mentioned (`topics/modules-intro.md:37`)
**Doc says:**
> It is also possible to load a module at runtime using the following command:
>
>     MODULE LOAD /path/to/mymodule.so

**Problem:** On a default 9.1.2 server, `MODULE LOAD` and `MODULE UNLOAD` are rejected. The page (lines 18-19 and 37-48) presents runtime loading as always available. `grep -rl enable-module-command topics/ commands/` finds no page in valkey-doc that mentions the setting.
**Evidence:** `valkey@9.1.2:src/config.c:3371`: `createEnumConfig("enable-module-command", NULL, IMMUTABLE_CONFIG, protected_action_enum, server.enable_module_cmd, PROTECTED_ACTION_ALLOWED_NO, ...)`. Live 9.1.2, default config:
```
$ valkey-cli -p 20900 module load /tmp/docreview-modules-intro/hw.so
ERR MODULE command not allowed. If the enable-module-command option is set to "local", you can run it from a local connection, otherwise you need to set this option in the configuration file, and then restart the server.
```
With `--enable-module-command local`, the same command returns `OK`.
**Fix:** After line 39 add: "`MODULE LOAD` and `MODULE UNLOAD` are disabled by default. To use them, set `enable-module-command` to `yes` (any connection) or `local` (loopback connections only) in `valkey.conf` and restart the server. The setting cannot be changed at runtime. The `loadmodule` directive works regardless of this setting."

### F2 [minor] INACCURATE: `ValkeyModule_Call` errno values are wrong (`topics/modules-intro.md:355`)
**Doc says:**
> NULL is returned when the command name is invalid, the format specifier uses characters that are not recognized, or when the command is called with the wrong number of arguments. In the above cases the `errno` var is set to `EINVAL`.

**Problem:** Only wrong arity sets `EINVAL`. An unknown command sets `ENOENT` and a bad format specifier sets `EBADF`. The page also leaves out the other errno values `Call` can set.
**Evidence:** `valkey@9.1.2:src/module.c` (VM_Call doc comment, about line 6527): "EBADF: wrong format specifier. EINVAL: wrong command arity. ENOENT: command does not exist. EPERM ... EROFS ... ENETDOWN ... ENOTSUP ... EACCES ... ENOSPC ... ESPIPE". Test module calling `Call(ctx,"NOSUCHCMD","c","x")`, `Call(ctx,"GET","Z","x")`, `Call(ctx,"GET","cc","x","y")` on 9.1.2 and returning errno:
```
2     (ENOENT)
9     (EBADF)
22    (EINVAL)
```
**Fix:** Replace with: "On failure NULL is returned and `errno` is set: `ENOENT` if the command does not exist, `EBADF` for an invalid format specifier, `EINVAL` for wrong arity, `EPERM` if a key hashes to a non-local slot in cluster mode. See [`ValkeyModule_Call`](modules-api-ref.md#ValkeyModule_Call) for the full list (`EROFS`, `ENETDOWN`, `EACCES`, `ENOSPC`, `ESPIPE`, `ENOTSUP`)."

### F3 [minor] INACCURATE: `Call()` commands are not emitted before `Replicate()` commands (`topics/modules-intro.md:881`)
**Doc says:**
> Commands replicated with `Call()` are always the first emitted in the final `MULTI/EXEC` block, while all the commands emitted with `Replicate()` will follow.

**Problem:** On 9.1.2, commands are propagated in the order they run. A module that relies on the documented order will replicate the wrong sequence. The `VM_Replicate` source comment repeats the stale claim, but the behavior does not follow it.
**Evidence:** Test command running `ValkeyModule_Replicate(ctx,"SET","cc","fromreplicate","1")` and then `ValkeyModule_Call(ctx,"SET","!cc","fromcall","1")`. The resulting 9.1.2 AOF:
```
*1 $5 MULTI  *3 $3 SET $13 fromreplicate $1 1  *3 $3 SET $8 fromcall $1 1  *1 $4 EXEC
```
**Fix:** Replace lines 881-885 with: "If you mix `Call()` replication and `Replicate()`, the commands are emitted inside the `MULTI/EXEC` block in the order they were executed." Also fix the `VM_Replicate` comment in `src/module.c`, because modules-api-ref.md is generated from it.

### F4 [minor] INACCURATE: format specifier list described as "full" is missing ten specifiers (`topics/modules-intro.md:341`)
**Doc says:**
> This is the full list of format specifiers:

**Problem:** The list covers `c b s l v ! A R`. It leaves out `3`, `0`, `C`, `S`, `W`, `M`, `E`, `D`, `K`, `X`. Some of these change security or correctness behavior: `C` applies ACL checks, `S` enforces script restrictions, `E` returns errors as replies. The list duplicates the generated API reference and has fallen behind it.
**Evidence:** `valkey@9.1.2:src/module.c` VM_Call comment (about lines 6447-6520) documents `3`, `0`, `C`, `S`, `W`, `M`, `E`, `D`, `K`, `X` in addition to the eight listed here.
**Fix:** Change the lead-in to "The most common format specifiers are:" and add after the list: "See [`ValkeyModule_Call`](modules-api-ref.md#ValkeyModule_Call) for the full list, including `C` (run with the context user's ACL permissions), `S` (script mode), `E` (return errors as reply objects), and `3` (RESP3 replies)." Treat the API reference as the canonical source.

### F5 [minor] INACCURATE: reply type and key type lists are incomplete (`topics/modules-intro.md:377`, `topics/modules-intro.md:659`)
**Doc says:**
> Valid reply types are: ... `VALKEYMODULE_REPLY_NULL` NULL reply.

> It returns one of the following values: ... VALKEYMODULE_KEYTYPE_ZSET
> The above are just the usual Valkey key types

**Problem:** Both lists are presented as exhaustive. A module that switches on them will mishandle RESP3 replies (from the `3` or `0` specifiers) and keys of stream or module type.
**Evidence:** `valkey@9.1.2:src/valkeymodule.h:97-98`: `VALKEYMODULE_KEYTYPE_MODULE 6`, `VALKEYMODULE_KEYTYPE_STREAM 7`. Lines 101-116: `VALKEYMODULE_REPLY_UNKNOWN -1`, `..._MAP 5`, `..._SET 6`, `..._BOOL 7`, `..._DOUBLE 8`, `..._BIG_NUMBER 9`, `..._VERBATIM_STRING 10`, `..._ATTRIBUTE 11`, `..._PROMISE 12`, `..._SIMPLE_STRING 13`, `..._ARRAY_NULL 14`.
**Fix:** Add `VALKEYMODULE_KEYTYPE_MODULE` and `VALKEYMODULE_KEYTYPE_STREAM` to the key type block. Introduce the reply types with "RESP2 replies use these types:" and add after the list: "RESP3 calls (`3` or `0` specifier) can also return `MAP`, `SET`, `BOOL`, `DOUBLE`, `BIG_NUMBER`, `VERBATIM_STRING`, and `ATTRIBUTE`. `PROMISE` is returned with `K`, and `SIMPLE_STRING` and `ARRAY_NULL` with `X`."

### F6 [minor] OUTDATED: hello world example uses the deprecated `random` flag (`topics/modules-intro.md:75`)
**Doc says:**
> HelloworldRand_ValkeyCommand, "fast random",

**Problem:** `random` has been ignored since Redis OSS 7.0. The first example teaches a flag that does nothing.
**Evidence:** `valkey@9.1.2:src/module.c:1266`: `else if (!strcasecmp(t,"random")) { /* Deprecated. Silently ignore. */ }`. Lines 1339-1341: "Starting from Redis OSS 7.0 this flag has been deprecated. Declaring a command as "random" can be done using command tips". `COMMAND INFO helloworld.rand` on 9.1.2 shows only the flags `module` and `fast`.
**Fix:** Use `"fast"` as the flags string. If nondeterminism matters, add one sentence: "To mark output as nondeterministic, set the `nondeterministic_output` [command tip](command-tips.md) with `ValkeyModule_SetCommandInfo`."

### F7 [minor] OUTDATED: compatibility section talks about "Redis OSS versions" (`topics/modules-intro.md:193`)
**Doc says:**
> A module can be designed to support both newer and older Redis OSS versions where certain API functions are not available in all versions.
> If an API function is not implemented in the currently running Redis OSS version, the function pointer is set to NULL.

**Problem:** The section is about Valkey versions and `valkeymodule.h`. "Redis OSS" here is a leftover from a search-and-replace, not compatibility guidance.
**Evidence:** The surrounding text (lines 184-191, 203) is about `valkeymodule.h` and Valkey versions. `RMAPI_FUNC_SUPPORTED` is defined in `valkey@9.1.2:src/valkeymodule.h:2721`.
**Fix:** Replace "Redis OSS" with "Valkey" in both sentences.

### F8 [minor] OUTDATED: implementation languages described in the future tense (`topics/modules-intro.md:20`)
**Doc says:**
> Modules are meant to be written in C, however it will be possible to use C++ or other languages that have C binding functionalities.

**Problem:** It is already possible, and Valkey's own modules prove it. valkey-json and valkey-search are C++, and valkey-bloom is Rust using the Valkey Rust SDK. Readers are not told this.
**Evidence:** `~/git/valkey-bloom` is a Rust crate (Cargo.toml, `valkey_module` dependency). `~/git/valkey-json/src` and `~/git/valkey-search/src` are C++.
**Fix:** "The API is a C API, so modules can be written in C or any language with C bindings. Official Valkey modules include C++ (valkey-json, valkey-search) and Rust (valkey-bloom, built with [valkeymodule-rs](https://github.com/valkey-io/valkeymodule-rs))."
Tracked: valkey-io/valkey-io.github.io#343 (Rust SDK on website, related)

### F9 [minor] MISSING: no build step and no source for `valkeymodule.h` (`topics/modules-intro.md:184`)
**Doc says:**
> In order to create a new module, just copy a recent version of `valkeymodule.h` in your source tree, link all the libraries you want, and create a dynamic library having the `ValkeyModule_OnLoad()` function symbol exported.

**Problem:** The page goes from the example source straight to loading a `.so`. It never says where the header lives or how to build the library, so a reader new to modules cannot get from line 61 to line 35.
**Evidence:** The header is at `valkey@9.1.2:src/valkeymodule.h`. The hello world example built and loaded on 9.1.2 with `gcc -fPIC -shared -o hw.so hw.c`.
**Fix:** After line 189 add: "The header is `src/valkeymodule.h` in the [Valkey repository](https://github.com/valkey-io/valkey/blob/unstable/src/valkeymodule.h). Build the example with:" followed by a `sh` block containing `gcc -fPIC -shared -o helloworld.so helloworld.c`. Consider moving this section to just after the example.

### F10 [minor] AI-NATIVE: the opening is a table of contents and never says what the page covers or who it is for (`topics/modules-intro.md:7`)
**Doc says:**
> The modules documentation is composed of the following pages:
> * Introduction to Valkey modules (this file). An overview about Valkey Modules system and API. It's a good idea to start your reading here.

and front matter `description: Introduction to writing Valkey modules`.
**Problem:** The first paragraph is navigation, and "(this file)" refers to the repo rather than the published page. What a module is and who this guide is for only appears at line 14. The description is too short to summarize the page.
**Fix:** Open with lines 14-22, then add: "This page is for C developers writing their first module. It covers loading modules, registering commands, calling commands, low-level key access, replication, and memory management." Move the page list below that under a heading such as "## Other module documentation", and change "(this file)" to "(this page)". Set the description to "How to write, build, and load a Valkey module in C: registering commands, accessing keys, replication, and memory management."

### F11 [nit] EXAMPLE: prototypes on the page do not match `valkeymodule.h` (`topics/modules-intro.md:931`)
**Doc says:**
> void ValkeyModule_Calloc(size_t nmemb, size_t size);

also line 419 `char *ptr = ValkeyModule_CallReplyStringPtr(reply,&len);` and line 471 `ValkeyModule_ReplyWithError(ValkeyModuleCtx *ctx, const char *err);` (no return type).
**Evidence:** `valkey@9.1.2:src/valkeymodule.h:1572`: `void *(*ValkeyModule_Calloc)(size_t nmemb, size_t size)`. Line 1729: `const char *(*ValkeyModule_CallReplyStringPtr)(...)`. Line 415 of the page itself says the pointer "is returned as a `const` pointer".
**Fix:** `void *ValkeyModule_Calloc(size_t nmemb, size_t size);`, `const char *ptr = ValkeyModule_CallReplyStringPtr(reply,&len);`, `int ValkeyModule_ReplyWithError(ValkeyModuleCtx *ctx, const char *err);`.

### F12 [nit] CONFUSING: the "Example usage" after `VALKEYMODULE_ERRORMSG_WRONGTYPE` does not use it (`topics/modules-intro.md:474`)
**Doc says:**
> There is a predefined error string for key of wrong type errors: VALKEYMODULE_ERRORMSG_WRONGTYPE
> Example usage:
> ValkeyModule_ReplyWithError(ctx,"ERR invalid arguments");

**Fix:** Move the generic example directly under the `ReplyWithError` prototype. Then show the constant with its own example: `ValkeyModule_ReplyWithError(ctx, VALKEYMODULE_ERRORMSG_WRONGTYPE);`.

### F13 [nit] STYLE: heading level jumps to H1 mid-page, and one heading ends with a period (`topics/modules-intro.md:206`)
**Doc says:**
> # Passing configuration parameters to Valkey modules

and line 360 `## Working with ValkeyModuleCallReply objects.`
**Evidence:** The live page renders a second `<h1 id="passing-configuration-parameters-to-valkey-modules">` under the page title.
**Fix:** Change line 206 to `##` and remove the trailing period on line 360.

### F14 [nit] EXAMPLE: indented code blocks have no language tag (`topics/modules-intro.md:35`)
**Doc says:** indented blocks at lines 35, 39, 43, 48, 213, 476, 661-666, 808-809, e.g.
>     loadmodule /path/to/mymodule.so

**Fix:** Convert them to fenced blocks: `text` for the config lines (35, 213), `text` or `bash` with `valkey-cli` for the commands (39, 43, 48), and `c` for the constants (476, 661, 808).

### F15 [nit] STYLE: typos and wrong function names (`topics/modules-intro.md:462`)
**Doc says / Fix:**
- 362 "`ValkeyModuleCall` returns" → "`ValkeyModule_Call` returns"
- 424 "ValkeyCallReply objects" → "ValkeyModuleCallReply objects"
- 462 "BUSY the sever is busy" → "BUSY the server is busy"
- 562 "Each call to `SetArray()`" → "Each call to `ValkeyModule_ReplySetArrayLength()`"
- 653 "In order to obtain the value of a key, use the `ValkeyModule_KeyType()`" → "To get the type of a key, use `ValkeyModule_KeyType()`"
- 670 "does not yet exists" → "does not exist yet"
- 746 "0 is returned by the function:" → "0 is returned."
- 815 "For a complete list of set type functions, see [Key API for List type]" → "list type functions"
- 963 "so it the overhead" → "so the overhead"
- 972 "for the following commands" → "for the following functions"

### F16 [nit] AI-NATIVE: commodity filler adds length with no information (`topics/modules-intro.md:303`)
**Doc says:**
> Also note that sometimes using the low level API is not harder compared to the higher level one.

also line 853 "If you use the above programming style, there are no problems." and line 519 "To return nested arrays is easy,".
**Fix:** Delete lines 303-304 and line 853. Change line 519 to "For nested arrays, call `ValkeyModule_ReplyWithArray()` again for the nested element, then emit its elements."
