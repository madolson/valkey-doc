VERDICT: NEEDS-WORK
COUNTS: blocker=3 minor=4 nit=4

# Modules and blocking commands (`modules-blocking-ops`)

https://valkey.io/topics/modules-blocking-ops/

The prose is mostly right, but the C examples have three bugs that crash or misbehave if copied: the `free_privdata` signature is wrong, a pointer is passed where a value is expected, and a function call is missing its parentheses. Fix the examples against `src/modules/helloblock.c` first.

## Findings

### F1 [blocker] INACCURATE: `free_privdata` callback signature is wrong (`topics/modules-blocking-ops.md:41`)
**Doc says:**
> `void (*free_privdata)(void*),`

and at line 183:
> `void free_privdata(void *privdata) {`
> `    ValkeyModule_Free(privdata);`

**Problem:** The callback takes `(ValkeyModuleCtx *, void *)`. A callback written as the page shows gets the ctx pointer as its only argument, so `ValkeyModule_Free(privdata)` frees the context instead of the private data. The result is a crash or heap corruption, and the private data leaks.
**Evidence:** valkey@9.1.2:src/valkeymodule.h:2051-2056
```
VALKEYMODULE_API ValkeyModuleBlockedClient *(*ValkeyModule_BlockClient)(ValkeyModuleCtx *ctx,
    ValkeyModuleCmdFunc reply_callback,
    ValkeyModuleCmdFunc timeout_callback,
    void (*free_privdata)(ValkeyModuleCtx *, void *),
    long long timeout_ms)VALKEYMODULE_ATTR;
```
valkey@9.1.2:src/modules/helloblock.c:56 `void HelloBlock_FreeData(ValkeyModuleCtx *ctx, void *privdata) {`. Compiling the page's code against the 9.1.2 header:
```
t.c:23:90: warning: passing argument 4 of 'ValkeyModule_BlockClient' from incompatible pointer type [-Wincompatible-pointer-types]
```
**Fix:** Line 41: `void (*free_privdata)(ValkeyModuleCtx *, void *),`. Lines 183-185:
```C
void free_privdata(ValkeyModuleCtx *ctx, void *privdata) {
    VALKEYMODULE_NOT_USED(ctx);
    ValkeyModule_Free(privdata);
}
```

### F2 [blocker] EXAMPLE: Reply sends the pointer, not the number (`topics/modules-blocking-ops.md:174`)
**Doc says:**
> `return ValkeyModule_ReplyWithLongLong(ctx,mynumber);`

(also at line 253)
**Problem:** `mynumber` is a `long *`. The client gets the heap address as its integer reply, not the random number.
**Evidence:** gcc against the 9.1.2 header:
```
t.c:19:51: warning: passing argument 2 of 'ValkeyModule_ReplyWithLongLong' makes integer from pointer without a cast [-Wint-conversion]
```
valkey@9.1.2:src/modules/helloblock.c:45 `return ValkeyModule_ReplyWithLongLong(ctx, *myint);`
**Fix:** Use `ValkeyModule_ReplyWithLongLong(ctx,*mynumber);` at lines 174 and 253.

### F3 [blocker] EXAMPLE: `IsBlockedTimeoutRequest` is never called, so the command never blocks (`topics/modules-blocking-ops.md:254`)
**Doc says:**
> `} else if (ValkeyModule_IsBlockedTimeoutRequest) {`

**Problem:** This tests the function pointer, which is always non-NULL, instead of calling the function. On the first invocation of the command the branch is taken, so the command replies Null right away and never blocks.
**Evidence:** valkey@9.1.2:src/module.c:9130 `int VM_IsBlockedTimeoutRequest(ValkeyModuleCtx *ctx) {` takes a ctx argument. The API is exposed as a function pointer (`VALKEYMODULE_GET_API`), which is non-NULL after `ValkeyModule_Init`.
**Fix:** `} else if (ValkeyModule_IsBlockedTimeoutRequest(ctx)) {`

### F4 [minor] INACCURATE: The page implies the module may skip `UnblockClient` after a timeout or disconnect (`topics/modules-blocking-ops.md:53`)
**Doc says:**
> `ms` is the timeout in milliseconds. When the timeout is reached, the timeout callback is called and the client is automatically aborted.

and lines 71-75:
> The `privdata` data will be freed automatically using the `free_privdata` callback when the client is unblocked. This is useful **since the reply callback may never be called** in case the client timeouts or disconnects

**Problem:** A reader concludes that a timed-out client is cleaned up automatically. It is not. The module must still call `ValkeyModule_UnblockClient` on every blocked client, or the handle leaks. The page never says this. It also never says that `timeout_ms` of 0 means no timeout (every example passes 0), or that `CLIENT UNBLOCK` triggers the timeout callback.
**Evidence:** valkey@9.1.2:src/module.c:8672-8674
```
 * 1. ValkeyModule_UnblockClient should be called for every blocked client,
 * even if client was killed, timed-out or disconnected. Failing to do so
 * will result in memory leaks.
```
valkey@9.1.2:src/module.c:8666-8667 `timeout_callback: called when the timeout is reached or if `CLIENT UNBLOCK` is invoked`. valkey@9.1.2:src/module.c:8385 `if (timeout_ms) {` (0 means no timeout is set).
**Fix:** Replace line 53 with: "`timeout_ms` is the timeout in milliseconds, or 0 for no timeout. When it expires, or when another client runs `CLIENT UNBLOCK` on it, the timeout callback replies to the client. The module must still call `ValkeyModule_UnblockClient()` on the handle afterwards, even if the client timed out or disconnected, or the handle leaks." Rename `ms` to `timeout_ms` at lines 51 and 53 to match the signature.

### F5 [minor] EXAMPLE: The blocking calls never pass `free_privdata`, and the single-function example passes the wrong callbacks (`topics/modules-blocking-ops.md:259`)
**Doc says:**
> `ValkeyModule_BlockClient(ctx,reply_func,timeout_func,NULL,0);`

(the same call appears at lines 88 and 218)
**Problem:** Line 178 says a `free_privdata` must be passed, but no example ever passes one, so the `ValkeyModule_Alloc` at line 150 leaks on every call. The single-function example at line 259 still registers `reply_func` and `timeout_func`, so the `IsBlockedReplyRequest`/`IsBlockedTimeoutRequest` branches never run. That defeats the point of the section.
**Evidence:** valkey@9.1.2:src/modules/helloblock.c:110 `ValkeyModule_BlockClient(ctx, HelloBlock_Reply, HelloBlock_Timeout, HelloBlock_FreeData, timeout);`
**Fix:** Lines 218 and 259 should pass `free_privdata`. Line 259 should be `ValkeyModule_BlockClient(ctx,Example_ValkeyCommand,Example_ValkeyCommand,free_privdata,0);`. After line 186, add: "and the blocking call becomes `ValkeyModule_BlockClient(ctx,reply_func,timeout_func,free_privdata,0);`".

### F6 [minor] EXAMPLE: The thread functions return nothing and the threads are never detached (`topics/modules-blocking-ops.md:101`)
**Doc says:**
> `void *threadmain(void *arg) { ... ValkeyModule_UnblockClient(bc,NULL); }`

(also at line 153)
**Problem:** A `void *` function with no return is undefined behavior if the value is used. gcc warns about it. Nothing ever joins or detaches the threads, so every command call leaks a thread's resources.
**Evidence:**
```
t.c:13:1: warning: control reaches end of non-void function [-Wreturn-type]
```
valkey@9.1.2:src/modules/helloblock.c:73 `return NULL;`
**Fix:** Add `return NULL;` at the end of both `threadmain` bodies. After each successful `pthread_create`, add `pthread_detach(tid);` (lines 91, 221, 262).

### F7 [minor] MISSING: Key-based blocking, disconnect callbacks, and where blocking is not allowed are not covered (`topics/modules-blocking-ops.md:18`)
**Doc says:**
> this documentation shows how the API works and describes a few patterns that can be used in order to model blocking commands.

**Problem:** The page opens with `BLPOP` as the motivating example, but it never mentions `ValkeyModule_BlockClientOnKeys`/`SignalKeyAsReady`, which is how a module builds a BLPOP-style command. It also leaves out `ValkeyModule_SetDisconnectCallback` and `ValkeyModule_BlockedClientMeasureTimeStart/End`, and it never says that `BlockClient` does not block inside MULTI, inside scripts, or when called from a keyspace notification. A module author hits all of these.
**Evidence:** valkey@9.1.2:src/module.c:8679-8692
```
 * There are some cases where ValkeyModule_BlockClient() cannot be used:
 * 1. If the client is executing a script.
 * 2. If the client is executing a MULTI block.
 * 3. If the client is a temporary module client.
 * 4. If the client is already blocked.
 * In cases 1 and 2, a call to ValkeyModule_BlockClient() will **not** block the
 * client, but instead produce a specific error reply.
```
The API exists at valkey@9.1.2:src/module.c:8847 (`VM_SignalKeyAsReady`), 8949 (`VM_SetDisconnectCallback`), and 2491 (`VM_BlockedClientMeasureTimeStart`).
**Fix:** Before "Thread safe contexts", add a section "Blocking on keys" with two paragraphs that cover `BlockClientOnKeys` plus `SignalKeyAsReady` and link to [modules-api-ref.md#section-blocking-clients-from-modules](modules-api-ref.md#section-blocking-clients-from-modules). After line 75, add a short list of when `BlockClient` does not block (MULTI, scripts, already-blocked clients), with the behavior quoted above. Also add one sentence each on `SetDisconnectCallback` and `BlockedClientMeasureTimeStart/End`.

### F8 [nit] OUTDATED: The external references point to Redis-era material or are not linked (`topics/modules-blocking-ops.md:285`)
**Doc says:**
> `[Neural Redis module](https://github.com/antirez/neural-redis)`

and line 26:
> You may want to check the `helloblock.c` example in the Valkey source tree inside the `src/modules` directory

**Problem:** neural-redis is a Redis module from antirez's personal repository, not maintained for Valkey. The `helloblock.c` reference, which is the best working example (it is correct where the page's own code is not), has no link.
**Evidence:** `curl` returns 200 for the neural-redis repo, but the project targets the Redis module API. valkey@9.1.2 `git ls-tree 9.1.2 src/modules/` lists `src/modules/helloblock.c`.
**Fix:** Line 26: "See [`src/modules/helloblock.c`](https://github.com/valkey-io/valkey/blob/unstable/src/modules/helloblock.c) for a complete, working example." Lines 284-287: either drop the paragraph, or reword it to "For an example of this pattern, see neural-redis, a Redis module whose networks train in background threads while the user queries older versions."

### F9 [nit] STYLE: Typos and wording errors throughout (pattern)
**Doc says:**
> line 81 `int his command, in order to take the example simple`; line 139 `actually expansive operation`; line 142 `as follow`; line 199 `in order to implement the non blocking command`; line 202 `We don't want to take the client blocked`; line 236 `implement the reply and callback`; line 62 `and unblock the client`; line 190 `disconnects or timeout`; line 271 `Functionally is the same`.

**Problem:** Line 199 says the opposite of what it means ("non blocking" where it means blocking). The rest are typos.
**Evidence:** Text as quoted.
**Fix:** "in this command, to keep the example simple"; "expensive"; "as follows"; "to implement the blocking command"; "We don't want to leave the client blocked"; "the reply and timeout callbacks"; "and unblocks the client"; "disconnects or times out"; "It is functionally the same".

### F10 [nit] STYLE: A heading ends with a period, and the voice is inconsistent (`topics/modules-blocking-ops.md:23`)
**Doc says:**
> `How blocking and resuming works.`

and line 244:
> `So I could rewrite the example command`

**Problem:** The rendered H2 includes the trailing period. The page switches between "we", "I", and "you".
**Evidence:** Live page: `<h2 id="how-blocking-and-resuming-works">How blocking and resuming works.`
**Fix:** Change the heading to "How blocking and resuming works" and line 244 to "The example command can be rewritten without separate reply and timeout callbacks:".

### F11 [nit] AI-NATIVE: The opening paragraph does not say who the page is for (`topics/modules-blocking-ops.md:7`)
**Doc says:**
> Valkey has a few blocking commands among the built-in set of commands.

**Problem:** It takes three paragraphs before the page says it is a C module API guide. Lines 11-16 are generic background.
**Evidence:** Lines 7-20 as written.
**Fix:** Replace lines 7-20 with: "This page is for module authors writing blocking commands in C. A blocking command, like `BLPOP`, suspends only the calling client, not the server, until an external event happens: a key changes, a background thread finishes, or a timeout expires. It covers `ValkeyModule_BlockClient`, unblocking with private data, aborting a block, and the threading patterns that go with them."
