VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=3

# Transactions (`transactions`)

https://valkey.io/topics/transactions/

Examples and error strings match 9.1.2. The AOF paragraph is wrong for the default config: with the default `aof-load-truncated yes`, Valkey reverts a partial transaction and starts, it does not exit. The page also leaves out two cluster/standalone rules that change behavior (CROSSSLOT and the REDIRECT capability).

## Findings

### F1 [minor] INACCURATE: Partial AOF transaction does not stop startup by default (`topics/transactions.md:25`)
**Doc says:**
> However if the Valkey server crashes or is killed by the system administrator in some hard way it is possible that only a partial number of operations are registered. Valkey will detect this condition at restart, and will exit with an error. Using the `valkey-check-aof` tool it is possible to fix the append only file ...
**Problem:** `aof-load-truncated` defaults to `yes`. When the AOF ends inside a MULTI, Valkey truncates the file back to the offset before MULTI, logs a warning, and loads. It only exits when `aof-load-truncated` is `no`.
**Evidence:** `valkey@9.1.2:src/config.c:3291`
```
createBoolConfig("aof-load-truncated", NULL, MODIFIABLE_CONFIG, server.aof_load_truncated, 1, NULL, NULL),
```
`valkey@9.1.2:src/aof.c:1725-1747`
```
uxeof: /* Unexpected AOF end of file. */
    if (fakeClient->flag.multi) {
        serverLog(LL_WARNING, "Revert incomplete MULTI/EXEC transaction in AOF file %s", filename);
        valid_up_to = valid_before_multi;
    }
    if (server.aof_load_truncated) {
        ...
                serverLog(LL_WARNING, "AOF %s loaded anyway because aof-load-truncated is enabled", filename);
                ret = AOF_TRUNCATED;
                goto loaded_ok;
```
**Fix:** Replace with: "If the server crashes or is killed partway through that write, the AOF can end with an incomplete transaction. On restart, with the default `aof-load-truncated yes`, Valkey discards the incomplete transaction, truncates the AOF to the last complete command, logs a warning, and starts. If `aof-load-truncated` is `no`, Valkey exits with an error, and you can run `valkey-check-aof --fix` to remove the partial transaction. See [Persistence](persistence.md#append-only-file)."

### F2 [minor] INACCURATE: EXEC reply described as a bulk string (`topics/transactions.md:106`)
**Doc says:**
> `EXEC` returned two-element [bulk string reply](protocol.md#bulk-strings) where one is an `OK` code and the other an error reply.
**Problem:** The reply is an array (`*2`), as the example directly above it shows. Its elements are a simple string and an error, not bulk strings.
**Evidence:** 9.1.2 live run:
```
EXEC
*2
+OK
-WRONGTYPE Operation against a key holding the wrong kind of value
```
**Fix:** "`EXEC` returned a two-element [array reply](protocol.md#arrays): a simple string `OK` and an error reply."

### F3 [minor] MISSING: `-REDIRECT` only goes to clients that opted in with `CLIENT CAPA redirect` (`topics/transactions.md:129`)
**Doc says:**
> If either event has occurred, a `-MOVED` or `-REDIRECT` error will be returned if needed without processing the transaction.
**Problem:** The standalone examples (lines 141-148, 161-167) show `-REDIRECT` as if every client gets it. The server sends it only to clients that sent `CLIENT CAPA redirect`. Other clients do not get `-REDIRECT`, so a reader who tests this will not see it.
**Evidence:** `valkey@9.1.2:src/server.c:3549`
```
return !server.cluster_enabled && server.primary_host && c->capa & CLIENT_CAPA_REDIRECT;
```
**Fix:** After line 130 add: "In standalone mode, `-REDIRECT` is sent only to clients that enabled it with [`CLIENT CAPA redirect`](../commands/client-capa.md)."

### F4 [minor] MISSING: Cluster rule that all keys in a transaction must be in one slot (`topics/transactions.md:132`)
**Doc says:**
> For cluster mode:
**Problem:** The cluster section covers MOVED but never states that every key across the queued commands must hash to the same slot. If they don't, `EXEC` fails with `-CROSSSLOT` and nothing runs. Cluster users hit this first.
**Evidence:** `valkey@9.1.2:src/cluster.c:1075-1078`
```
            } else if (c->mstate->commands[i].slot != -1 && c->mstate->commands[i].slot != slot) {
                if (error_code) *error_code = CLUSTER_REDIR_CROSS_SLOT;
                return NULL;
            }
```
**Fix:** Before line 129 add: "In cluster mode, all keys used by the commands in a transaction must hash to the same slot. Use [hash tags](cluster-spec.md#hash-tags) to co-locate them. Otherwise `EXEC` returns a `-CROSSSLOT` error and no command is executed."

### F5 [nit] MISSING: Nesting errors for MULTI and WATCH are not stated (`topics/transactions.md:256`)
**Doc says:**
> `WATCH` can be called multiple times.
**Problem:** A reader could take this to mean `WATCH` works after `MULTI`. It doesn't, and nested `MULTI` is also rejected. Neither is on the page.
**Evidence:** 9.1.2 live run:
```
MULTI      -> +OK
WATCH x    -> -ERR Command 'watch' not allowed inside a transaction
MULTI      -> -ERR Command 'multi' not allowed inside a transaction
```
**Fix:** Append: "`WATCH` must be called before `MULTI`. Calling `WATCH` or `MULTI` inside a transaction returns an error."

### F6 [nit] BROKEN: `#nil-reply` anchor lands on the Arrays heading, not on null arrays (`topics/transactions.md:201`)
**Doc says:**
> `EXEC` returns a [Null reply](protocol.md#nil-reply)
**Problem:** `protocol.md:216` places `<a name="nil-reply"></a>` right above `### Arrays`. A failed `EXEC` returns a null array (`*-1` in RESP2, `_` in RESP3, confirmed live), which is documented under `#### Null arrays`. Same link at line 288.
**Fix:** Change both links (lines 201 and 288) to `protocol.md#null-arrays`.

### F7 [nit] AI-NATIVE: Vague headings, a stub section, and a scripting link that goes to a command page (`topics/transactions.md:37`)
**Doc says:**
> ## Usage
> ## WATCH explained
> ## What about rollbacks?
> [scripts](../commands/eval.md) which are transactional.
**Problem:** "Usage" and "WATCH explained" don't say what the section answers. The rollbacks section is one sentence that gives no practical consequence. The scripting section links the EVAL command page instead of the scripting topic.
**Fix:** Rename "Usage" to "Running a transaction with MULTI and EXEC" and "WATCH explained" to "How WATCH decides whether EXEC runs". In the rollbacks section add: "Commands that already ran in a failed transaction keep their effects. Validate inputs before `EXEC`, or use a script if you need conditional logic." Link scripts to [Scripting with Lua](eval-intro.md) and [Functions](functions-intro.md).
