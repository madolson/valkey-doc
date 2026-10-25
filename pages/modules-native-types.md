VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=5 nit=5

# Modules API for native types (`modules-native-types`)

https://valkey.io/topics/modules-native-types/

Redis 4.0-era tutorial lightly renamed to Valkey. It says `digest` and `mem_usage` are unsupported (both work in 9.1.2), documents 5 of the ~20 type callbacks, has an empty AOF section, and its examples don't compile as written. Fix first: delete the "not supported" claims and link to the `ValkeyModule_CreateDataType` reference for the full callback list.

## Findings

### F1 [minor] OUTDATED: `digest` and `mem_usage` described as unsupported (`topics/modules-native-types.md:71`)
**Doc says:**
> while `.digest` and `.mem_usage` are optional and are currently not actually supported by the modules internals, so for now you can just ignore them.

and line 124:
> `digest` is called when `DEBUG DIGEST` is executed ... Currently this is not yet implemented so the function ca be left empty.

**Problem:** Both callbacks are called in 9.1.2. `DEBUG DIGEST` calls `digest`, and `MEMORY USAGE` calls `mem_usage`. A reader who follows the page and skips them gets `MEMORY USAGE` returning a meaningless size and digests that leave out module keys.
**Evidence:** valkey@9.1.2:src/debug.c:271-277:
```
    } else if (o->type == OBJ_MODULE) {
        ...
        if (mt->digest) {
            mt->digest(&md, mv->value);
```
valkey@9.1.2:src/module.c:13002-13003:
```
    } else if (mt->mem_usage != NULL) {
        size = mt->mem_usage(mv->value);
```
Live 9.1.2 with `src/modules/hellotype.c` loaded (it sets `.mem_usage`):
```
> hellotype.insert k 5
1
> hellotype.insert k 7
2
> memory usage k
72
```
**Fix:** Replace lines 70-73 with: "The methods above are required. `digest` (used by `DEBUG DIGEST`) and `mem_usage` (used by `MEMORY USAGE`) are optional but recommended." On line 124, delete "Currently this is not yet implemented so the function ca be left empty."

### F2 [minor] MISSING: Callback list is incomplete and the `mem_usage` prototype is wrong (`topics/modules-native-types.md:108`)
**Doc says:**
> `rdb_load`, `rdb_save`, `aof_rewrite`, `digest` and `free` and `mem_usage` are all callbacks with the following prototypes
> `typedef size_t (*ValkeyModuleTypeMemUsageFunc)(void *value);`

**Problem:** 9.1.2 is at `VALKEYMODULE_TYPE_METHOD_VERSION 5` and also accepts `aux_load`, `aux_save`, `aux_save2`, `free_effort`, `unlink`, `copy`, `defrag`, `mem_usage2`, `free_effort2`, `unlink2` and `copy2`. The page never mentions them. Without `copy`, `COPY` on a module key fails. Without `defrag`, active defrag skips the module's values. `mem_usage` actually takes `const void *`, so the prototype on the page produces an incompatible-pointer warning, and an error on compilers that treat that warning as an error.
**Evidence:** valkey@9.1.2:src/valkeymodule.h:62 `#define VALKEYMODULE_TYPE_METHOD_VERSION 5`; :1480 `typedef size_t (*ValkeyModuleTypeMemUsageFunc)(const void *value);`; :1477-1490 list the other callback typedefs. valkey@9.1.2:src/module.c:7370-7432 documents every optional field.
**Fix:** Change line 116 to `typedef size_t (*ValkeyModuleTypeMemUsageFunc)(const void *value);`. After the bullet list at line 126, add: "Optional callbacks for lazy free, `COPY`, active defrag, and auxiliary (out-of-keyspace) RDB data are listed in the [`ValkeyModule_CreateDataType` reference](modules-api-ref.md#ValkeyModule_CreateDataType)."

### F3 [minor] MISSING: "AOF rewriting" section has no content (`topics/modules-native-types.md:368`)
**Doc says:**
> AOF rewriting
> ---
> ```C
> void ValkeyModule_EmitAOF(ValkeyModuleIO *io, const char *cmdname, const char *fmt, ...);
> ```

**Problem:** The section is just a bare prototype. It doesn't explain that `aof_rewrite` must emit commands that rebuild the key, what `fmt` means, or that `aof_rewrite` is called unconditionally. A NULL `aof_rewrite` crashes the server during `BGREWRITEAOF`.
**Evidence:** valkey@9.1.2:src/aof.c:2327 `mt->aof_rewrite(&io, key, mv->value);` (no NULL check). valkey@9.1.2:src/modules/hellotype.c:279 `ValkeyModule_EmitAOF(aof, "HELLOTYPE.INSERT", "sl", key, node->value);`
**Fix:** Add under the prototype: "`aof_rewrite` receives the key name and value and must call `ValkeyModule_EmitAOF()` once per command needed to rebuild the value. `cmdname` is one of the module's own commands. `fmt` uses the same specifiers as `ValkeyModule_Call()` (`s` for a `ValkeyModuleString*`, `l` for `long long`, `c` for a C string, `b` for buffer plus length). Example from hellotype.c:" followed by the `HelloTypeAofRewrite` function.

### F4 [minor] INACCURATE: RDB I/O API list is incomplete and the "no error checking" claim is unqualified (`topics/modules-native-types.md:287`)
**Doc says:**
> Valkey offers a high level API that can automatically store inside the RDB file the following types: Unsigned 64 bit integers. Signed 64 bit integers. Doubles. Strings.
> The functions don't require any error checking from the module, that can always assume calls succeed.

**Problem:** 9.1.2 also has `SaveFloat`/`LoadFloat` and `SaveLongDouble`/`LoadLongDouble`. "No error checking" is only true by default, where a read error aborts the server. Modules that set `VALKEYMODULE_OPTIONS_HANDLE_IO_ERRORS` (needed for diskless loading with `repl-diskless-load swapdb`) must call `ValkeyModule_IsIOError()` after reads.
**Evidence:** valkey@9.1.2:src/valkeymodule.h:1861-1864 (`SaveFloat`, `LoadFloat`, `SaveLongDouble`, `LoadLongDouble`); :321 `#define VALKEYMODULE_OPTIONS_HANDLE_IO_ERRORS (1 << 0)`; :1843 `ValkeyModule_IsIOError`. topics/modules-api-ref.md:971: "The module should use `ValkeyModule_IsIOError` after reads".
**Fix:** Add "Floats" and "Long doubles" to the list and their four functions to the code block. Replace lines 315-316 with: "By default a read or write failure aborts the server, so the module doesn't need to check results. A module that sets `VALKEYMODULE_OPTIONS_HANDLE_IO_ERRORS` via [`ValkeyModule_SetModuleOptions`](modules-api-ref.md#ValkeyModule_SetModuleOptions) must check `ValkeyModule_IsIOError()` after loading."

### F5 [minor] EXAMPLE: Code samples don't compile (`topics/modules-native-types.md:251`)
**Doc says:**
> line 251: `ValkeyModule_ModuleTypeSetValue(key,MyTyke,data);`
> line 332: `struct dobule_array *da = ptr;`
> lines 53-65: `int ValkeyModule_OnLoad(ValkeyModuleCtx *ctx) { ... if (MyType == NULL) return VALKEYMODULE_ERR; }`
> line 345: `if (encver != DOUBLE_ARRAY_ENC_VER) {`

**Problem:** `MyTyke` and `dobule_array` are typos. The second one makes `da->count` an error, because `struct dobule_array` is an incomplete type. `OnLoad` has no success return, which is undefined behavior: the server reads garbage and may reject the module. Its signature is also missing the `argv`/`argc` parameters that hellotype.c uses, and its first body lines aren't indented. `DOUBLE_ARRAY_ENC_VER` is never defined.
**Evidence:** valkey@9.1.2:src/modules/hellotype.c:322-324 calls `ValkeyModule_CreateDataType(ctx, "hellotype", 0, &tm);` then `if (HelloType == NULL) return VALKEYMODULE_ERR;` and ends `OnLoad` with `return VALKEYMODULE_OK;`.
**Fix:** `MyTyke` → `MyType`; `dobule_array` → `double_array`; add `return VALKEYMODULE_OK;` before the closing brace at line 65 and use `int ValkeyModule_OnLoad(ValkeyModuleCtx *ctx, ValkeyModuleString **argv, int argc)`, indenting the body consistently; add `#define DOUBLE_ARRAY_ENC_VER 0` above `DoubleArrayRDBLoad`.

### F6 [nit] BROKEN: Wrong path to hellotype.c, not linked (`topics/modules-native-types.md:37`)
**Doc says:**
> is available inside the Valkey distribution in the `/modules/hellotype.c` file.

**Problem:** The file is at `src/modules/hellotype.c`.
**Evidence:** `git -C ~/git/valkey ls-tree -r --name-only 9.1.2 | grep hellotype` prints `src/modules/hellotype.c`.
**Fix:** "...in [`src/modules/hellotype.c`](https://github.com/valkey-io/valkey/blob/unstable/src/modules/hellotype.c)."

### F7 [nit] STYLE: Heading levels out of order (`topics/modules-native-types.md:41`)
**Doc says:**
> Registering a new data type
> ===

**Problem:** The `===` underline renders as a second `<h1>` between `<h2>` sections. The live page shows `h2 Overview of native types`, then `h1 Registering a new data type`, then `h2 ...`. Line 128's heading "Ok, but *why* modules types require a 9 characters name?" is chatty, and it's really a subsection of registration.
**Fix:** Change line 42 to `---`. Retitle line 128 "Why type names are exactly 9 characters" and make it `###`.

### F8 [nit] STYLE: Code blocks without a language tag (`topics/modules-native-types.md:212`)
**Doc says:**
> line 212: ```` ``` ```` (fenced, untagged); lines 138 and 163: indented blocks.

**Problem:** These are the only code samples on the page without a language tag.
**Fix:** Tag line 212 as `C`. Make lines 138 and 163 fenced `text` blocks.

### F9 [nit] STYLE: Conversational first-person voice and typos (`topics/modules-native-types.md:131`)
**Doc says:**
> "Oh, I understand you need to understand this" (131); "if I'm building a *b-tree* data structure and my name is *antirez*" (87); "So that she or he immediately realizes" (177); "ca be left empty" (124); "as a first glance" (34); "take in mind" (403); "Certain are optionals" (70); "Everything is pretty simple." (152); "writing a new data structure for Valkey is a simple task" (34)

**Problem:** It's the original author's first-person voice, along with typos and filler that adds nothing.
**Fix:** Delete line 131-132 and lines 152, 34's "so in practical terms..." clause. Change line 87 to "For example, a b-tree type by author *antirez* could be named `btree1-az`." Line 177 → "so the user immediately sees which module is missing." Fix the typos: "can", "at first glance", "keep in mind", "Some are optional".

### F10 [nit] MISSING: No links to the API reference or modules intro (`topics/modules-native-types.md:379`)
**Doc says:**
> (see the other Valkey Modules documentation for detailed information).

**Problem:** It points to "other documentation" without a link. The page has no links to modules-intro.md or modules-api-ref.md anywhere, even though modules-api-ref.md:3968 links back to this page.
**Fix:** "(see [`ValkeyModule_Alloc`](modules-api-ref.md#ValkeyModule_Alloc) in the API reference)". Add to the first paragraph: "This page assumes familiarity with [the modules introduction](modules-intro.md); every function is documented in the [Modules API reference](modules-api-ref.md)."
