VERDICT: NEEDS-WORK
COUNTS: blocker=1 minor=7 nit=3

# Modules API reference (`modules-api-ref`)

https://valkey.io/topics/modules-api-ref/

API coverage matches 9.1.2 exactly (all 373 registered APIs documented except `GetContextFromIO`), and the anchors all resolve. The page was not regenerated after 9.1.2, so it is missing the `ScriptingEngineDebuggerLog` ownership rule that prevents a double free.

## Findings

### F1 [blocker] OUTDATED: Page not regenerated after 9.1.2 and missing the `ScriptingEngineDebuggerLog` ownership rule (`topics/modules-api-ref.md:7938`)
**Doc says:**
> - `msg`: the message to send.

**Problem:** When I regenerate from 9.1.2 src/module.c, I get two doc-comment changes the page does not have. The important one: 9.1.2 states that ownership of `msg` passes to the debugger log. A module that frees `msg` after the call, which is the normal pattern for every other API on this page, gets a use-after-free or double free. The server had this same bug in its own Lua debugger (#4574). The second change is the RegisterClusterMessageReceiver note at line 5288: it is missing the type-255 compatibility note and the closing period.
**Evidence:** valkey@9.1.2:src/module.c:14555-14556:
```
 * - `msg`: the message to send. Ownership of `msg` is transferred to the
 *   debugger log. The caller must not free it or access it after this call.
```
valkey@9.1.2:src/module.c:9619-9620:
```
 * Note: Old versions of Valkey could not handle type 255. This was fixed in
 * 9.1.2, 9.0.6, 8.1.10, 8.0.11 and 7.2.15.
```
The diff of 9.1.2 `utils/generate-module-api-doc.rb` output against the page shows only these two content hunks. Everything else differs only in the "Available since" lines, because the generator ran without tags.
**Fix:** Regenerate topics/modules-api-ref.md from 9.1.2 (`utils/generate-module-api-doc.rb`) and commit it. Note: the 9.1.2 generator fails on current Ruby (`undefined method '+@' for "/../src/module.c"` at line 229, from `File.dirname(__FILE__) ++ "..."`). Change `++` to `+` in the script.

### F2 [minor] INACCURATE: `ValkeyModuleEvent_ReplBackup` and its subevents are documented but not defined in valkeymodule.h (`topics/modules-api-ref.md:7344`)
**Doc says:**
> * `ValkeyModuleEvent_ReplBackup`
> WARNING: Replication Backup events are deprecated since Redis OSS 7.0 and are never fired.
> ...
> * `VALKEYMODULE_SUBEVENT_REPL_BACKUP_CREATE` / `_RESTORE` / `_DISCARD`

**Problem:** Valkey's header no longer defines the event struct or the three subevent macros. Code that copies these names does not compile. The entry documents nothing a Valkey module can use.
**Evidence:** valkey@9.1.2:src/valkeymodule.h:531: `#define VALKEYMODULE_EVENT_REPL_BACKUP 12 /* Not used anymore. */`. The header has no `ValkeyModuleEvent_ReplBackup` and no `VALKEYMODULE_SUBEVENT_REPL_BACKUP_*`: `grep -c SUBEVENT_REPL_BACKUP valkeymodule.h` returns 0. The doc text comes from valkey@9.1.2:src/module.c:12508-12523.
**Fix:** Delete the `ValkeyModuleEvent_ReplBackup` bullet from the VM_SubscribeToServerEvent comment (src/module.c:12508-12524). For history, replace it with one line under ReplAsyncLoad: "ReplBackup (event 12) was removed. Use ReplAsyncLoad."

### F3 [minor] MISSING: `GetContextFlags` omits two flags that 9.1.2 sets (`topics/modules-api-ref.md:2225`)
**Doc says:** The flag list at lines 2225-2288 runs from `VALKEYMODULE_CTX_FLAGS_LUA` to `VALKEYMODULE_CTX_FLAGS_SLOT_EXPORT_CLIENT`. It has no `DENY_BLOCKING` and no `ASYNC_LOADING`.
**Problem:** The function sets both flags. `DENY_BLOCKING` is the flag a module needs to decide whether it may call `BlockClient`, so this gap matters.
**Evidence:** valkey@9.1.2:src/module.c:4126: `if (ctx->client->flag.deny_blocking) flags |= VALKEYMODULE_CTX_FLAGS_DENY_BLOCKING;` and :4153: `flags |= VALKEYMODULE_CTX_FLAGS_ASYNC_LOADING;`. valkey@9.1.2:src/valkeymodule.h:218-224 defines both.
**Fix:** Add to the list in the VM_GetContextFlags comment (src/module.c, near 4109):
```
 *  * VALKEYMODULE_CTX_FLAGS_DENY_BLOCKING: The current client does not allow
 *    blocking (called from MULTI, Lua, or another module via ValkeyModule_Call).
 *  * VALKEYMODULE_CTX_FLAGS_ASYNC_LOADING: The server is loading a dataset
 *    asynchronously during diskless replication (repl-diskless-load swapdb).
```

### F4 [minor] MISSING: `ValkeyModule_GetContextFromIO` is exported but has no entry (`topics/modules-api-ref.md:4490`)
**Doc says:** The "IO context handling" section starts directly with `ValkeyModule_GetKeyNameFromIO`. `GetContextFromIO` appears nowhere on the page.
**Problem:** It is the only one of the 373 APIs registered in 9.1.2 with no entry. It has no doc comment, so the generator skips it.
**Evidence:** valkey@9.1.2:src/module.c:8121: `ValkeyModuleCtx *VM_GetContextFromIO(ValkeyModuleIO *io) {`, :15337: `REGISTER_API(GetContextFromIO);`. valkey@9.1.2:src/valkeymodule.h:1889 declares it.
**Fix:** Add a comment above VM_GetContextFromIO in src/module.c: "Return a ValkeyModuleCtx for use inside rdb_load, rdb_save, aof_rewrite, and similar type callbacks that receive only a ValkeyModuleIO. The context is owned by the IO and is freed with it. Do not free it."

### F5 [minor] CONFUSING: "Available since" shows Redis OSS versions with no note saying so (`topics/modules-api-ref.md:71`)
**Doc says:**
> **Available since:** 4.0.0

**Problem:** 339 of the 373 entries list a version before 7.2.4 (106 say 4.0.0, 67 say 6.0.0, and so on). Valkey has no 4.0.0 or 6.0.0. A reader cannot tell that these are Redis OSS versions and that everything below 7.2.4 means "in every Valkey release".
**Evidence:** `grep 'Available since' topics/modules-api-ref.md | sort | uniq -c`: 4.0.0 x106, 5.0.0 x38, 6.0.0 x67, 6.2.0 x20, 7.0.0 x62, 7.2.0 x20, then 8.0.0 x6, 8.1.0 x5, 9.1.0 x12.
**Fix:** In utils/generate-module-api-doc.rb, add to the generated header: "Versions earlier than 7.2.4 refer to Redis OSS releases. Every API marked with such a version is present in all Valkey releases." Alternatively, render those versions as "Redis OSS 4.0.0".

### F6 [minor] AI-NATIVE: No intro, weak description, and page opens on a bare list of 45 links (`topics/modules-api-ref.md:11`)
**Doc says:**
> description: >
>     Reference for the Valkey Modules API
> ...
> ## Sections

**Problem:** The page has no sentence saying what it covers, who it is for, or where to start. Someone arriving from search lands on a list of section links and has to find modules-intro alone. "Sections" is a heading that carries no meaning.
**Evidence:** topics/modules-api-ref.md:1-13. The generator writes this header (valkey@9.1.2:utils/generate-module-api-doc.rb:227).
**Fix:** In the generator, emit before the list: "This page lists every C function a Valkey module can call through `valkeymodule.h`, generated from the comments in `src/module.c`. Each entry gives the signature, the release that introduced it, and its behavior. New to modules? Start with [Modules API introduction](modules-intro.md)." Rename `## Sections` to `## API sections`. Set description to "C function reference for Valkey modules: commands, keys, replies, replication, blocking, cluster, ACL, configs, and server events, with the version each API was added."

### F7 [minor] INACCURATE: `ValkeyModuleCommandResultInfo` field listing drops `version` and `rejection_context` (`topics/modules-api-ref.md:7482`)
**Doc says:**
> const char *command_name;        // Full command name (e.g., "SET", "CLIENT|LIST")
> ...
> ValkeyModuleString **argv;       // Command arguments (read-only, zero-copy)

**Problem:** The real struct starts with `uint64_t version` and ends with `rejection_context`. A reader who takes the listing as the layout gets the field offsets wrong. The prose at line 7518 also refers to `rejection_context`, which the listing does not show.
**Evidence:** valkey@9.1.2:src/valkeymodule.h:875-884:
```c
typedef struct ValkeyModuleCommandResultInfo {
    uint64_t version;              /* Version of this structure for ABI compat. */
    const char *command_name;
    ...
    ValkeyModuleString **argv;
    const char *rejection_context; /* Context string; meaning depends on the event: ...
```
**Fix:** In the comment at src/module.c:12645-12655, add `uint64_t version;  // Structure version` as the first line and `const char *rejection_context;  // Rejected/ACLRejected only, else NULL` as the last.

### F8 [nit] CONFUSING: KeyAtPos deprecation sentence names the wrong API (`topics/modules-api-ref.md:233`)
**Doc says:**
> This API existed before `ValkeyModule_KeyAtPosWithFlags` was added, now deprecated and can be used for compatibility with older versions

**Problem:** Grammatically, "now deprecated" attaches to KeyAtPosWithFlags, which is the current API.
**Fix:** In src/module.c:1164: "Deprecated. Use ValkeyModule_KeyAtPosWithFlags. This older form remains for compatibility with modules written before key-specs and key flags."

### F9 [nit] INACCURATE: Config names may contain underscores (`topics/modules-api-ref.md:7610`)
**Doc says:**
> The name must only contain alphanumeric characters or dashes.

**Evidence:** valkey@9.1.2:src/module.c:13862-13872, `moduleVerifyResourceName` accepts `(curr_char == '_') || (curr_char == '-')`.
**Fix:** In src/module.c:14113: "The name may contain only alphanumeric characters, underscores, and dashes."

### F10 [nit] STYLE: Doc-comment typos and inconsistencies across the page (`topics/modules-api-ref.md:7936`)
**Doc says / Fix:**
- 7936 (module.c:14553): "`ValkeyModule_VM_ScriptingEngineDebuggerFlushLogs`" → "`ValkeyModule_ScriptingEngineDebuggerFlushLogs`".
- 2278, 2283, 2286 (module.c:4109, 4114, 4117): "Indicate the that client" → "The client".
- 7614 (module.c:14117): "can only be provided loading time" → "can only be set at load time".
- 7617 (module.c:14120): "will be only be modifiable" → "is modifiable only".
- 3870, 3874, 3902 (module.c Call comment, e.g. 6524): `'D'`, `'K'`, `'X'` use single quotes. The rest of the list uses backticks (`` `D` ``). These render as plain text on the live page.
- 7475, 7486, 7497 (module.c:12639, 12650, 12661): "RM_Call" → "ValkeyModule_Call". The generator rewrites only `VM_`.

## Unverified
- topics/modules-api-ref.md:3907-3920: the `ValkeyModule_Call` errno list (EBADF, EPERM, ESPIPE, and so on). I confirmed all format specifiers against the parser but did not trace each errno path in 9.1.2.
