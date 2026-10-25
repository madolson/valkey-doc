VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=5 nit=4

# Migration from Redis to Valkey (`migration`)

https://valkey.io/topics/migration/

The migration procedures work, but the compatibility list has two factual errors for 9.x: Valkey 9 writes RDB files Redis cannot load, and the Lua version globals it names do not exist. Fix the RDB claim first, since readers rely on it to plan a rollback.

## Findings

### F1 [minor] INACCURATE: Valkey 9.x does not write Redis 7.2-compatible RDB files (`topics/migration.md:38`)
**Doc says:**
> Persistence: Valkey reads and writes RDB and AOF files compatible with Redis OSS 7.2.

**Problem:** Since 9.0, Valkey writes RDB version 80 with a `VALKEY` magic string. Redis 7.2 (RDB 11) refuses this file. Anyone who plans a rollback by copying a Valkey `dump.rdb` (or an AOF with an RDB preamble) back to Redis will find it does not work. Replicating to a Redis 7.2 replica still works, because the primary downgrades the stream to RDB 11 for replicas that do not report a version.

**Evidence:** valkey@9.1.2:src/rdb.h:46-58:
```
 * We start using high rdb version numbers since Valkey 9.0. ...
#define RDB_VERSION 80
    {11, 0x070200},
    {80, 0x090000},
```
Live 9.1.2 after `SAVE`:
```
00000000: 5641 4c4b 4559 3038 30fa 0a76 616c 6b65  VALKEY080..valke
```
valkey@9.1.2:src/replication.c:235: "Fallback to RDB 11, which was introduced in 7.2. 7.2 and older don't report their version."

**Fix:** "Persistence: Valkey reads RDB and AOF files produced by Redis OSS 7.2 and earlier. Valkey 7.2 and 8.x also write RDB version 11, which Redis 7.2 can load. Valkey 9.0 and later write RDB version 80, which Redis cannot load, so after you migrate to Valkey 9 you cannot roll back by copying the RDB file. A Valkey 9 primary still sends RDB 11 to Redis 7.2 replicas during replication."

### F2 [minor] INACCURATE: Lua `SERVER_VERSION` and `SERVER_VERSION_NUM` globals do not exist (`topics/migration.md:42`)
**Doc says:**
> Valkey also supports the `server` namespace and `SERVER_NAME`, `SERVER_VERSION`, and `SERVER_VERSION_NUM` globals.

**Problem:** None of these are globals. They are fields of the `server` and `redis` tables, and the version fields are called `VALKEY_VERSION` and `VALKEY_VERSION_NUM`. A script that reads `SERVER_NAME` as a global raises an error.

**Evidence:** valkey@9.1.2:src/modules/lua/script_lua.c:1705-1714 pushes `VALKEY_VERSION_NUM`, `VALKEY_VERSION`, and `SERVER_NAME` into the table. Live 9.1.2:
```
> eval 'return {SERVER_NAME, SERVER_VERSION, SERVER_VERSION_NUM}' 0
ERR user_script:1: Script attempted to access nonexistent global variable 'SERVER_NAME'
server.SERVER_NAME: valkey
server.SERVER_VERSION: nil
server.SERVER_VERSION_NUM: nil
server.VALKEY_VERSION -> 9.1.2 ; server.VALKEY_VERSION_NUM -> 590082
```
**Fix:** "Lua scripting: Existing scripts that use the `redis` namespace continue to work. Valkey also provides the `server` namespace, with the fields `server.SERVER_NAME`, `server.VALKEY_VERSION`, and `server.VALKEY_VERSION_NUM`. See [Lua API](lua-api.md)." (topics/lua-api.md:470 and :496 document the same wrong names and need the same fix.)

### F3 [minor] MISSING: no mention of `rdb-version-check relaxed` for Redis CE 7.4+ RDB files (`topics/migration.md:25`)
**Doc says:**
> Redis Community Edition (CE) version 7.4 and later produce data files that are not compatible with Valkey.

**Problem:** Since 8.1, Valkey has a `rdb-version-check relaxed` option. With it, Valkey will try to load RDB versions in the foreign range (12 to 79) and aborts only if it meets data it does not recognize. Users who hit "Can't handle RDB format version 12" never learn about this option.

**Evidence:** valkey@9.1.2:src/rdb.c:89-92:
```
    if (server.rdb_version_check == RDB_VERSION_CHECK_STRICT) {
        if (rdbver > RDB_VERSION) return false; /* future version */
        if (rdbIsForeignVersion(rdbver)) return false;
```
valkey@9.1.2:src/rdb.h:62-63 defines `RDB_FOREIGN_VERSION_MIN 12` and `RDB_FOREIGN_VERSION_MAX 79`. The default is `strict` (valkey.conf:614). It was added in commit e9b8970e7, first tagged 8.1.0.

**Fix:** After line 26, add: "Valkey 8.1 and later can try to load these files if you set `rdb-version-check relaxed`. This is best effort. Loading fails if the file uses data types or encodings that Valkey does not support. The setting also applies to replication and `RESTORE`." Change the table row at line 33 to "Valkey 8.1+ with `rdb-version-check relaxed` (best effort)".
Tracked: valkey-io/valkey-doc#419

### F4 [minor] OUTDATED: examples and links are pinned to Valkey 7.2 (`topics/migration.md:53`)
**Doc says:**
> The example migration steps are provided for Redis 7.2.5 and Valkey version 7.2.6.

**Problem:** The current release is 9.1.2, and the 7.2 framing appears across the page. Line 334 links to the `7.2` branch of valkey.conf. The Valkey `INFO KEYSPACE` output at lines 131 and 264 is missing a field that 9.1.2 prints. The `INFO REPLICATION` output at lines 205-207 is also missing a field that 9.1.2 prints.

**Evidence:** Live 9.1.2:
```
# Keyspace
db0:keys=2,expires=0,avg_ttl=0,keys_with_volatile_items=0
# Replication
role:master
connected_slaves:0
replicas_waiting_psync:0
```
**Fix:** Change line 53 to "The examples use Redis 7.2 and Valkey 9.1." Update the Valkey output blocks to match the 9.1.2 output above. Point line 334 at `https://raw.githubusercontent.com/valkey-io/valkey/9.1/valkey.conf`, or link to [valkey.conf](valkey.conf.md).

### F5 [minor] MISSING: the replication steps ignore Redis authentication (`topics/migration.md:157`)
**Doc says:**
> Connect to your Valkey container and start the `valkey-cli` to configure replication using the [REPLICAOF](../commands/replicaof.md) command.

**Problem:** If the Redis primary has `requirepass` or ACL users, `REPLICAOF` returns OK but the link stays down. Step 4 then shows `master_link_status:down` and the page gives no next step. The MIGRATE section states its no-auth assumption (line 231). This section does not.

**Evidence:** Live 9.1.2 replica with a link that is not established:
```
role:slave
master_link_status:down
master_last_io_seconds_ago:-1
```
`masterauth` and `masteruser` are the configs for this (valkey@9.1.2:valkey.conf, replication section).

**Fix:** Before `REPLICAOF` in step 3, add: "If Redis requires a password, set it on Valkey first: `CONFIG SET masterauth <password>`. If Redis uses ACLs, also run `CONFIG SET masteruser <username>`."

### F6 [nit] CONFUSING: step numbers in the source skip and repeat (`topics/migration.md:194`)
**Doc says:**
> 9. In your Valkey container, stop the Valkey replication ...

**Problem:** The replication steps go 1-5, 9, 10 (lines 194, 200). The MIGRATE steps go 1-4, 7, 4 (lines 288, 290). The live page renumbers them in order, so the rendered output is correct, but the source is wrong and easy to break.

**Evidence:** Lines 194, 200, 288, 290 as quoted.

**Fix:** Renumber to 6, 7 (lines 194, 200) and 5, 6 (lines 288, 290).

### F7 [nit] EXAMPLE: CLI prompts and container names are inconsistent (`topics/migration.md:72`)
**Doc says:**
> redis 127.0.0.1:6379> INFO KEYSPACE

and
> $ docker exec -it somevalkey valkey-cli

**Problem:** Neither CLI prints a `redis ` or `valkey ` prefix. The prompt is `127.0.0.1:6379>`, as line 203 shows. Line 121 names the container `valkey-container-name`, but line 128 then runs `somevalkey`, and line 387 runs `valkey-container-name`. Code blocks throughout have no language tag (lines 70, 81, 90, 120, 127, 144, 150, 159, 167, 187, 196, 202, 237, 254, 261, 269, 274, 292, 324, 336, 347, 359, 365, 372, 378, 386, 401, 412, 418).

**Evidence:** valkey-cli 9.1.2 prompt seen in every live session: `127.0.0.1:20840>`.

**Fix:** Drop the `redis `/`valkey ` prompt prefix. Use one container name per role (`myredis`, `myvalkey`). Tag shell blocks `bash`, tag CLI sessions and outputs `text`, and tag the config block at line 336 `text`.

### F8 [nit] STYLE: legacy flag and non-inclusive prose (`topics/migration.md:413`)
**Doc says:**
> --cluster-replica --cluster-master-id node-id

and line 322: "three primary (master) and three replica (slave) nodes."

**Problem:** valkey-cli accepts `--cluster-primary-id` (valkey@9.1.2:src/valkey-cli.c:2740). In prose, the terms in parentheses add nothing. The literal output already uses them, and lines 218 and 424 explain why.

**Evidence:** valkey@9.1.2:src/valkey-cli.c:2740: `!strcmp(argv[i], "--cluster-master-id") || !strcmp(argv[i], "--cluster-primary-id")`.

**Fix:** Use `--cluster-primary-id node-id` at line 413. Change line 322 to "three primary and three replica nodes." Drop "(master)" at line 408 and "(slave)" at line 376. Remove the duplicate note at line 218 and keep the one at line 424.

### F9 [nit] AI-NATIVE: headings that are vague or ungrammatical (`topics/migration.md:12`)
**Doc says:**
> ## Why to migrate to Valkey?

and line 136: `### Replication`

**Problem:** "Why to migrate" is ungrammatical. "Replication" names a mechanism, not the task, and it collides with the general replication topic.

**Evidence:** Headings as quoted.

**Fix:** Rename to `## Why migrate to Valkey` and `### Migrate with replication (minimal downtime)`. Rename `### Physical migration` to `### Migrate by copying the RDB file`.

## Unverified
- topics/migration.md:310-422: that a Valkey 9.1.2 node can join a Redis 7.2 cluster bus, replicate from it, and take over with `CLUSTER FAILOVER`. Not tested, because there was no Redis binary. The source shows the primary sends RDB 11 to replicas that report no version (src/replication.c:235), so leftover Redis replicas should be able to full-sync from a promoted Valkey 9 primary. Cluster bus compatibility was not checked.
