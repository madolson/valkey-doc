VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=7 nit=6

# Replication (`replication`)

https://valkey.io/topics/replication/

The newer sections (full sync, dual-channel, persistence safety) are accurate for 9.1.2. The older text still carries Redis-era claims that 9.1.2 contradicts. The most important fix: expired and evicted keys reach replicas as `UNLINK`, not `DEL`, and the page says `DEL` three times.

## Findings

### F1 [minor] INACCURATE: Expires and evictions propagate as UNLINK, not DEL (`topics/replication.md:303`)
**Doc says:**
> When a key expires on the primary, this is propagated as a [`DEL`](../commands/del.md) command.

Also line 396 ("it synthesizes a `DEL` command which is transmitted to all the replicas") and line 468 ("sending the DEL commands to the replica as keys evict").
**Problem:** `lazyfree-lazy-expire` and `lazyfree-lazy-eviction` both default to `yes`, so the primary propagates `UNLINK`. Anyone watching the replication stream or AOF for `DEL` will not find it.
**Evidence:** valkey@9.1.2:src/db.c:2007 `argv[0] = lazy ? shared.unlink : shared.del;`. valkey@9.1.2:src/config.c:3278-3279 both lazyfree configs default `1`. Live 9.1.2, `MONITOR` on the replica after `SET k v PX 100` on the primary:
```
"set" "k" "v" "PXAT" "1792802994687"
"UNLINK" "k"
```
**Fix:** Line 303: "When a key expires on the primary, the deletion is propagated as an [`UNLINK`](../commands/unlink.md) (or `DEL` if `lazyfree-lazy-expire` is `no`)." Lines 396 and 468: replace "`DEL`" with "`UNLINK` (or `DEL`)".

### F2 [minor] INACCURATE: Serving stale data is the default, and loading does not block connections (`topics/replication.md:38`)
**Doc says:**
> it can handle queries using the old version of the dataset, assuming you configured Valkey to do so in valkey.conf. [...] The replica will block incoming connections during this brief window
**Problem:** `replica-serve-stale-data` defaults to `yes`, so this is not opt-in. The same is true of background deletion of the old dataset (`replica-lazy-flush` defaults to `yes`), which the page presents as something "you can configure". While it loads the new dataset, the replica still accepts connections and rejects commands with `-LOADING`. It does not block connections. The page also never names the config.
**Evidence:** valkey@9.1.2:src/config.c:3296 `createBoolConfig("replica-serve-stale-data", ..., server.repl_serve_stale_data, 1, ...)`. valkey@9.1.2:src/config.c:3295 `createBoolConfig("replica-lazy-flush", ..., server.repl_replica_lazy_flush, 1, ...)`. valkey@9.1.2:src/server.c:4588-4589 `if (server.loading && !server.async_loading && is_denyloading_command) { rejectCommand(c, shared.loadingerr, 1);`
**Fix:** "While the replica performs the initial synchronization it serves queries from the old dataset by default (`replica-serve-stale-data yes`). Set it to `no` to return an error instead. After the transfer, the replica must drop the old dataset and load the new one. During that window, which can last many seconds for large datasets, it replies `-LOADING` to most commands. By default (`replica-lazy-flush yes`) the old dataset is freed in a background thread, but loading still runs on the main thread. `repl-diskless-load swapdb` keeps serving the old data during the transfer."

### F3 [minor] INACCURATE: DEBUG is disabled by default (`topics/replication.md:295`)
**Doc says:**
> because administrative commands like [`DEBUG`](../commands/debug.md) or [`CONFIG`](../commands/config.md) are still enabled.
**Problem:** `enable-debug-command` defaults to `no`, so DEBUG is not available out of the box.
**Evidence:** valkey@9.1.2:src/config.c:3370 `createEnumConfig("enable-debug-command", ..., PROTECTED_ACTION_ALLOWED_NO, ...)`. Live 9.1.2 replica:
```
ERR DEBUG command not allowed. If the enable-debug-command option is set to "local", ...
```
**Fix:** "because administrative commands like [`CONFIG`](../commands/config.md), [`REPLICAOF`](../commands/replicaof.md) and [`FLUSHALL`](../commands/flushall.md) are still enabled."

### F4 [minor] OUTDATED: 9.2-only compression behavior described in the Full synchronization section without a version label (`topics/replication.md:187`)
**Doc says:**
> This happens when the replica asks for a filtered dataset, needs an older RDB version, or needs a different compression codec than the one the file is written in.

Also line 192 ("a different compression codec"). Lines 258-262 tell readers to match `repl-compression` with `rdbcompression`.
**Problem:** Compression codecs exist only in 9.2 (9.2.0-rc1 and unstable). In 9.1.2, `rdbcompression` is a yes/no bool and `repl-compression` does not exist. The Compressing section is labeled "Available since Valkey 9.2", but the Full synchronization section is not, so 9.1.2 readers are told about a fallback that cannot happen.
**Evidence:** valkey@9.1.2:src/config.c:3273 `createBoolConfig("rdbcompression", ...)` and no `repl-compression` in 9.1.2. valkey@9.2.0-rc1:src/config.c:3538,3540 add the enum `rdbcompression` and `repl-compression`. valkey@9.1.2:src/replication.c:1002-1004 lists only the filter and RDB-version conditions:
```
socket_target = (mincapa & REPLICA_CAPA_EOF) && (server.repl_diskless_sync ||
                                                 (req & REPLICA_REQ_RDB_MASK) ||
                                                 rdbver != RDB_VERSION);
```
**Fix:** Line 187: "...asks for a filtered dataset, needs an older RDB version, or (since Valkey 9.2) needs a different compression codec than the one the file is written in." Make the same edit at line 192. At line 262, add "(`rdbcompression` accepts codec names since Valkey 9.2)".
Tracked: valkey-io/valkey-doc#487

### F5 [minor] MISSING: Replica authentication covers only requirepass, not ACL users (`topics/replication.md:342`)
**Doc says:**
> If your primary has a password via `requirepass`, it's trivial to configure the replica to use that password in all sync operations.
**Problem:** Deployments that use ACLs need `primaryuser` as well. Without it the replica authenticates as `default`, and the sync fails or runs with the wrong permissions. The section never mentions `primaryuser`.
**Evidence:** valkey@9.1.2:src/config.c:3329 `createStringConfig("primaryuser", "masteruser", ...)`.
**Fix:** Append: "If the primary uses [ACLs](acl.md), also set `primaryuser <username>`. The user needs at least `+psync +replconf +ping`." Retitle the section "Authenticating a replica to its primary".

### F6 [minor] DUPLICATE: Failover replication-ID handling is explained twice on the page (`topics/replication.md:443`)
**Doc says:**
> When an instance is promoted to primary after a failover, it will still be able to perform a partial resynchronization with the replicas of the old primary. [...] using the same replication ID in the promoted replica would violate the rule
**Problem:** Lines 443-453 restate lines 130-148 (secondary ID kept after promotion, new ID because the old primary may still accept writes). Readers read the same mechanism twice in different words.
**Evidence:** Line 136 "it sets its secondary ID to its main ID" and lines 144-148 "retaining the same replication ID would violate the fact that the same ID and same offset..." match lines 445-453.
**Fix:** Delete lines 443-453. Start the section with one line: "After a failover, replicas of the old primary can partially resync with the new one (see [Replication ID explained](#replication-id-explained))." Retitle it "Partial sync after restarts".

### F7 [minor] MISSING: Backlog sizing, the main knob for avoiding full syncs, has no names or defaults (`topics/replication.md:279`)
**Doc says:**
> There are also a few parameters for tuning the replication backlog taken in memory by the primary to perform the partial resynchronization. See the example `valkey.conf` shipped with the Valkey distribution for more information.
**Problem:** Full resyncs after short disconnects are the most common replication problem in production. The page never names `repl-backlog-size` or `repl-backlog-ttl` or gives their defaults. Line 381 sends readers to valkey.conf again for the same reason.
**Evidence:** valkey@9.1.2:src/config.c:3463 `"repl-backlog-size", ... 10 * 1024 * 1024` (10mb). valkey@9.1.2:src/config.c:3487 `"repl-backlog-ttl", ... 60 * 60` (1 hour).
**Fix:** "Partial resynchronization works only if the bytes the replica missed are still in the primary's backlog. `repl-backlog-size` (default `10mb`) sets its size: make it at least your write throughput in bytes per second times the longest disconnect you want to survive without a full sync. `repl-backlog-ttl` (default `3600` seconds) frees the backlog after the primary has had no replicas for that long." Under min-replicas (line 377), state the defaults: `min-replicas-to-write 0` (disabled), `min-replicas-max-lag 10`.

### F8 [nit] OUTDATED: Redis version numbers presented as if they were Valkey versions (`topics/replication.md:315`)
**Doc says:**
> As of version 7.0, these use cases are now all obsolete

Also line 329 ("Before version 4.0, writable replicas were incapable of expiring keys") and line 332 ("even in versions greater than 4.0").
**Problem:** Valkey started at 7.2. Readers can't tell these are Redis versions, and the 4.0 history doesn't apply to any Valkey release.
**Fix:** Delete line 329 and the version clause on line 332 ("Note though that using `EXPIRE` on a key that could ever exist on the primary..."). Line 315: "These use cases are now obsolete and can be achieved by other means:".

### F9 [nit] INACCURATE: REPLICAOF wording says the primary starts the sync (`topics/replication.md:271`)
**Doc says:**
> Alternatively, you can call the [`REPLICAOF`](../commands/replicaof.md) command and the primary host will start a sync with the replica.
**Problem:** The replica connects to the primary and sends `PSYNC` (line 96 says so). The primary does not start the sync.
**Fix:** "Alternatively, run [`REPLICAOF`](../commands/replicaof.md) on the replica at runtime; it connects to the primary and starts the sync."

### F10 [nit] CONFUSING: "evict" used where "expire" is meant, and an unexplained 63 limit (`topics/replication.md:331`)
**Doc says:**
> Valkey is able to evict keys with TTL as primaries do, with the exceptions of keys written in DB numbers greater than 63
**Problem:** This is about writable replicas expiring keys they wrote locally, not about eviction.
**Evidence:** valkey@9.1.2:src/expire.c:620 `if (db->id > 63) return;` in the replica-keys-with-expire tracking.
**Fix:** "A writable replica expires keys that it wrote locally with a TTL, except in databases numbered above 63."

### F11 [nit] STYLE: Indented code blocks with no language tag (`topics/replication.md:268`)
**Doc says:**
>     replicaof 192.168.1.1 6379
**Problem:** Every code sample is an indented block with no language tag: lines 92, 268, 336, 347, 351, 420-421, 478.
**Fix:** Use fenced blocks: ```` ```conf ```` for the config lines (268, 351, 420-421, 478), ```` ```text ```` for 92 and 336, and show 347 as a full command in ```` ```bash ````: `valkey-cli -h <replica> CONFIG SET primaryauth <password>`.

### F12 [nit] AI-NATIVE: "The INFO and ROLE command" section names the commands but no fields (`topics/replication.md:431`)
**Doc says:**
> There are two Valkey commands that provide a lot of information on the current replication parameters
**Problem:** The section gives no fields a reader can quote or monitor. The heading is also ungrammatical.
**Fix:** Retitle it "Monitoring replication with INFO and ROLE". List the fields operators check: `role`, `master_link_status`, `master_sync_in_progress`, `slave_repl_offset` vs the primary's `master_repl_offset`, `connected_slaves`, and per-replica `lag`. Link [INFO](../commands/info.md) and [ROLE](../commands/role.md).

### F13 [nit] STYLE: Grammar errors across the older sections (`topics/replication.md:10`)
**Doc says:**
> When a primary and a replica instances are well-connected

Also line 87 "that it is produced to be sent", line 302 "Changes on the primary is replicated", line 332 "could ever exists", lines 462-463 "before shutting down it, than can be restarted", line 476 "The configuration directives to use is".
**Fix:** "a primary and a replica are well connected". "that it produces to send". "Changes on the primary are replicated". "could ever exist". "before shutting it down, then restarted". "The configuration directive to use is".
