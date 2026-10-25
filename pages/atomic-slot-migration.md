VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=3 nit=4

# Atomic slot migration (`atomic-slot-migration`)

https://valkey.io/topics/atomic-slot-migration/

The page is accurate on 9.1.2. What I checked live (FLUSHALL fails the migration, importing keys hidden from DBSIZE/KEYS/SCAN/INFO, config names) matches. The main gap is operational: it never says when `CLUSTER MIGRATESLOTS` refuses to start, and it names thresholds without giving their defaults.

## Findings

### F1 [minor] MISSING: No prerequisites or start-time refusal conditions (`topics/atomic-slot-migration.md:15`)
**Doc says:**
> To perform an atomic slot migration, an operator performs the following steps:
**Problem:** The page doesn't say that the command must be sent to a primary, that it is refused while any legacy `CLUSTER SETSLOT` migration is in progress, or that every loaded module must declare atomic slot migration support. An operator with a loaded module that lacks support, or with a leftover MIGRATING/IMPORTING slot, gets an error the page never mentions. The linked command page doesn't cover these either.
**Evidence:** valkey@9.1.2:src/cluster_migrateslots.c:517-533
```
int validateSlotMigrationCanStartOrReply(client *c) {
    if (moduleVerifyAllAllowAtomicSlotMigrationOrReply(c) == C_ERR) {
    ...
        addReplyError(c, "Slot migration can only be used on primary nodes.");
    ...
        addReplyError(c, "Slots are being manually imported");
    ...
        addReplyError(c, "Slots are being manually migrated");
```
valkey@9.1.2:src/module.c:7631: `"The module %s does not support atomic slot migrations. Please ensure all modules have declared support for atomic slot migration and try again"`
**Fix:** Add before the steps: "Prerequisites: send the command to the primary that owns the slots. It is rejected while any slot is in a legacy `CLUSTER SETSLOT` MIGRATING or IMPORTING state, and if any loaded module has not declared atomic slot migration support. The caller needs ACL access to all databases (`alldbs`)."

### F2 [minor] AI-NATIVE: Thresholds and configs referenced without names or defaults (`topics/atomic-slot-migration.md:35`, `:71`, `:112`, `:117`)
**Doc says:**
> recently completed migrations will be visible up to a configurable threshold

> Once the amount of in-flight mutations goes below a configured threshold

> `slot-migration-max-failover-repl-bytes`: By default, atomic slot migration will only proceed to pausing mutations ... once all in-flight mutations have been sent
**Problem:** Lines 35 and 71 don't name the config. The configuration section gives no default values and leaves out that `-1` disables the limit. Readers have to open valkey.conf to get numbers they would want to quote.
**Evidence:** valkey@9.1.2:src/config.c:3451 `createULongConfig("cluster-slot-migration-log-max-len", ..., 0, LONG_MAX, ..., 1000, ...)`; src/config.c:3484 `createSSizeTConfig("slot-migration-max-failover-repl-bytes", ..., -1, SSIZE_MAX, ..., 0, MEMORY_CONFIG ...)`; valkey.conf:2057 "Setting this to -1 will disable this limit, and 0 will require no data be in the source output buffer". Live 9.1.2 `CONFIG GET *slot-migration*` returned 1000 and 0.
**Fix:** Line 35: "up to `cluster-slot-migration-log-max-len` entries (default 1000)". Line 71: "below `slot-migration-max-failover-repl-bytes`". In the config section, add "(default 0: pause only when the source output buffer is empty; -1 disables the limit; progress is visible in the `remaining_repl_size` field of `CLUSTER GETSLOTMIGRATIONS`)" and "(default 1000)" for the log length.

### F3 [minor] CONFUSING: "both the source and target node" reads as a conjunction (`topics/atomic-slot-migration.md:97`)
**Doc says:**
> `FLUSHDB` and `FLUSHALL` present a special case where we fail the slot migration when being executed on **both the source and target node**.
**Problem:** Read literally, the migration fails only if the flush runs on both nodes. In fact a flush on either node fails every in-progress job on that node. Live run: `FLUSHALL` on the source alone failed the export.
**Evidence:** valkey@9.1.2:src/cluster_migrateslots.c:961-970 (`clusterHandleFlushDuringSlotMigration` fails all in-progress jobs with "Data was flushed"). Live `CLUSTER GETSLOTMIGRATIONS` on the source after `FLUSHALL` on the source only:
```
state
failed
message
Data was flushed
```
**Fix:** "Running `FLUSHDB` or `FLUSHALL` on either the source or the target node fails all in-progress slot migrations on that node with the message `Data was flushed`. Retry the migration after the flush. It completes almost instantly because the slots are now empty."

### F4 [nit] CONFUSING: "sent to the whole cluster" omits the error from non-source nodes (`topics/atomic-slot-migration.md:46`)
**Doc says:**
> This command can be sent to the whole cluster to cancel all slot migrations everywhere.
**Problem:** Nodes with no active export reply with an error, so a broadcast returns errors from most nodes. Readers may read those as failures.
**Evidence:** valkey@9.1.2:src/cluster_migrateslots.c:1282-1284 `if (!clusterIsAnySlotExporting()) { addReplyError(c, "No migrations ongoing");`. Live: `ERR No migrations ongoing`.
**Fix:** Append: "Nodes with no outgoing migration reply `ERR No migrations ongoing`, which is expected."

### F5 [nit] INACCURATE: Snapshot does not iterate all hash slots (`topics/atomic-slot-migration.md:63`)
**Doc says:**
> The fork iterates all hash slots and serializes their contents
**Problem:** The child process serializes only the slots in the migration job.
**Evidence:** valkey@9.1.2:src/cluster_migrateslots.c:1560-1564 "Predicate function supplied to rewriteAppendOnlyFileRio to filter to only slots in this migration."
**Fix:** "The child iterates the migrating hash slots and serializes their contents".

### F6 [nit] STYLE: Typos and passive or first-person phrasing (`topics/atomic-slot-migration.md:9`, `:75`, `:97`, `:110`)
**Doc says:**
> introduced a option

> broadcasts ownership to the cluster

> where we fail the slot migration

> configured large enough to accumulate the accumulated mutations.
**Problem:** "a option" is a typo, line 75 has no final period, "we" is unclear about who acts, and line 110 repeats itself.
**Fix:** "introduced an option"; add the period at line 75; "Valkey fails the slot migration"; "large enough to hold the mutations accumulated during the snapshot."

### F7 [nit] AI-NATIVE: Weak front matter description (`topics/atomic-slot-migration.md:3`)
**Doc says:**
> description: Overview of atomic slot migration
**Problem:** The description just repeats the title and says nothing about what the page covers or who it is for.
**Fix:** `description: How to move hash slots between Valkey Cluster nodes with CLUSTER MIGRATESLOTS (Valkey 9.0+), monitor and cancel migrations, how it works internally, and which configs to tune.`
