VERDICT: NEEDS-WORK
COUNTS: blocker=1 minor=10 nit=5

# Cluster specification (`cluster-spec`)

https://valkey.io/topics/cluster-spec/

The design narrative mostly holds up, but several version-specific additions and examples are stale or wrong for 9.1.2. Most important fix: `CLUSTER SETSLOT ... TIMEOUT` is in milliseconds, not seconds.

## Findings

### F1 [blocker] INACCURATE: CLUSTER SETSLOT TIMEOUT is milliseconds, not seconds (`topics/cluster-spec.md:546`)
**Doc says:**
> The `timeout` is specified in seconds, where a value of 0 indicates an indefinite wait time.
**Problem:** The server parses TIMEOUT as milliseconds. An operator who sets `TIMEOUT 10` expecting 10 seconds gets 10 ms, and the command fails with NOREPLICAS. `commands/cluster-setslot.md:96` has the same error.
**Evidence:** `valkey@9.1.2:src/cluster_legacy.c:7766`:
```c
if (getTimeoutFromObjectOrReply(c, c->argv[i + 1], &timeout, UNIT_MILLISECONDS) != C_OK) return 0;
```
Live 9.1.2, 3x3 cluster, replica of the primary SIGSTOPped:
```
NOREPLICAS Not enough good replicas to write.
[timeout 1500] elapsed ms 1542
NOREPLICAS Not enough good replicas to write.
[timeout 1] elapsed ms 5
NOREPLICAS Not enough good replicas to write.
[] elapsed ms 2005
```
**Fix:** "The `timeout` is specified in milliseconds (default 2000), where a value of 0 indicates an indefinite wait time." Apply the same fix to `commands/cluster-setslot.md:96`.

### F2 [minor] OUTDATED: Replica election delay formula predates 9.1 (`topics/cluster-spec.md:885`)
**Doc says:**
> DELAY = 500 milliseconds + random delay between 0 and 500 milliseconds +
>         REPLICA_RANK * 1000 milliseconds.
**Problem:** Since 9.1.0 the fixed delay scales with node timeout (#2449), a failed-primary rank term was added, and the best-ranked replica skips the delay (#2227). The formula holds only at the default 15000 ms timeout and omits the other terms.
**Evidence:** `valkey@9.1.2:src/cluster_legacy.c:5657,5703-5733`:
```c
long long delay = min(server.cluster_node_timeout / 30, 500);
...
server.cluster->failover_auth_time = now + delay + (delay ? random() % delay : 0);
...
server.cluster->failover_auth_time += server.cluster->failover_auth_rank * (delay * 2);
...
if (getNodeReplicationOffset(myself) == 0) { server.cluster->failover_auth_time += 500; }
...
server.cluster->failover_auth_time += server.cluster->failover_failed_primary_rank * delay;
...
if (server.cluster->mf_end == 0 && myselfIsBestRankedReplica()) { server.cluster->failover_auth_time = now;
```
Both commits first appear in tag 9.1.0.
**Fix:** Replace the block with:
```
BASE  = min(NODE_TIMEOUT / 30, 500) milliseconds
DELAY = BASE + random(0, BASE) + REPLICA_RANK * 2 * BASE + FAILED_PRIMARY_RANK * BASE
```
Then add: "A replica with replication offset 0 adds 500 ms. Starting with Valkey 9.1, a replica that knows it is the best-ranked replica starts the election immediately. With the default `cluster-node-timeout` of 15000, BASE is 500 ms."

### F3 [minor] OUTDATED: CLUSTER SLOTS called deprecated (`topics/cluster-spec.md:394`)
**Doc says:**
> using the `CLUSTER SHARDS`, or the deprecated `CLUSTER SLOTS`, command
**Problem:** Valkey un-deprecated CLUSTER SLOTS in 8.0. The page itself recommends it at line 632, and `commands/cluster-slots.md` does not mark it deprecated.
**Evidence:** `valkey@3dd2f5a58` "Undeprecate cluster slots command (#536)", first in tag 8.0.0. `valkey@9.1.2:src/commands/cluster-slots.json` contains no `deprecated_since` or `replaced_by`.
**Fix:** "using the `CLUSTER SHARDS` or `CLUSTER SLOTS` command".

### F4 [minor] OUTDATED: "no need to specify a database other than 0" contradicts multi-database clusters (`topics/cluster-spec.md:522`)
**Doc says:**
> In Valkey Cluster there is no need to specify a database other than 0, but
> `MIGRATE` is a general command that can be used for other tasks not
> involving Valkey Cluster.
**Problem:** Since 9.0, a cluster can have `cluster-databases` > 1 (line 32 says so). Legacy resharding then has to migrate every database, and valkey-cli does this.
**Evidence:** `valkey@9.1.2:src/valkey-cli.c`, `clusterManagerMigrateKeysInSlot` (around line 5198): `int dbnum = getDatabases(source->context); ... config.conn_info.input_dbnum = 0; ... config.conn_info.input_dbnum++;`, which loops GETKEYSINSLOT/MIGRATE over each DB.
**Fix:** "If `cluster-databases` is greater than 1 (Valkey 9.0+), keys in the slot must be migrated from every database. `valkey-cli --cluster reshard` iterates over all databases and passes each one as `destination-db`."

### F5 [minor] EXAMPLE: CLUSTER NODES sample output uses the pre-Redis-4 format (`topics/cluster-spec.md:287`)
**Doc says:**
> d1861060fe6a534d42d8a19aeb36600e18785e04 127.0.0.1:6379 myself - 0 1318428930 1 connected 0-1364
>
> In the above listing the different fields are in order: node id, address:port, flags, last ping sent, last pong received, configuration epoch, link state, slots.
**Problem:** 9.1.2 prints `ip:port@cport`, shows `myself,master` and not just `myself`, and has a primary-ID field (`-`) that the field list leaves out. The listed order puts the wrong labels on columns 4 to 8.
**Evidence:** Live 9.1.2:
```
e5ad62e68d9ddfcb3bd1618da204d207964a6198 127.0.0.1:20240@30240 myself,master - 0 0 1 connected 0-5460
ca6336b67d1fb253c2a0355c5519cd1568712225 127.0.0.1:20241@30241 master - 0 1792802747448 2 connected 5461-10922
bca83955309fe4825561d632a8e5d0ddb28ca4cd 127.0.0.1:20244@30244 slave e5ad62e68d9ddfcb3bd1618da204d207964a6198 0 1792802747000 1 connected
```
**Fix:** Replace the sample with real 9.1.2 output like the above and change the field list to: "node id, ip:port@cport[,hostname], flags, primary node id (`-` for primaries), last ping sent, last pong received, configuration epoch, link state, slots." Link to `commands/cluster-nodes.md` for the full format.

### F6 [minor] EXAMPLE: CLUSTER SLOTS sample omits node ID and metadata; replica-hiding rule incomplete (`topics/cluster-spec.md:637`)
**Doc says:**
> 3) 1) "127.0.0.1"
>    2) (integer) 7001
>
> Replicas will be listed only when not in an error condition (i.e., when their FAIL flag is not set).
**Problem:** Since 4.0 each node entry includes the node ID, and since 7.0 a metadata map (9.1 adds `availability-zone`). Replicas are also hidden when their replication offset is 0, for example while they do the initial sync.
**Evidence:** Live 9.1.2:
```
1) 1) (integer) 0
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 20240
      3) "e5ad62e68d9ddfcb3bd1618da204d207964a6198"
      4) (empty array)
   4) 1) "127.0.0.1"
      2) (integer) 20244
      3) "bca83955309fe4825561d632a8e5d0ddb28ca4cd"
      4) (empty array)
```
Right after `--cluster create`, the same command listed no replicas for the first two ranges. `valkey@9.1.2:src/cluster.c:1476-1485`:
```c
if (clusterNodeIsFailing(node)) { return 0; }
/* Hide empty replicas in here, from a data-path POV, an empty replica is not available. */
return getNodeReplicationOffset(node) != 0;
```
**Fix:** Replace the sample with 9.1.2 output that includes IDs and metadata. Change line 663 to: "Replicas are listed only when they are not flagged FAIL and have a non-zero replication offset." Link to `commands/cluster-slots.md` for the entry format.

### F7 [minor] CONFUSING: Two different scale limits (`topics/cluster-spec.md:16`)
**Doc says:**
> High performance and linear scalability up to 2000 nodes.

and at line 133:
> however, the suggested max size of nodes is on the order of ~ 1000 nodes
**Problem:** The goals section was updated to 2000 nodes and the key distribution section was not, so a reader gets two answers to "how big can a cluster be".
**Evidence:** Lines 16 and 133 quoted above.
**Fix:** Line 133: "(however, the suggested max size is on the order of 2000 nodes)."

### F8 [minor] EXAMPLE: MIGRATE syntax has a phantom argument (`topics/cluster-spec.md:515`)
**Doc says:**
> MIGRATE target_host target_port "" target_database id timeout KEYS key1 key2 ...
**Problem:** "target_database id" reads as two arguments. The real syntax has one `destination-db` followed by `timeout`.
**Evidence:** `valkey@9.1.2:src/cluster.c:428`: `MIGRATE host port "" dbid timeout [COPY | REPLACE | AUTH password | ...] KEYS key1 ... keyN`.
**Fix:** `MIGRATE target_host target_port "" destination-db timeout KEYS key1 key2 ...`

### F9 [minor] DUPLICATE: SETSLOT replication section repeats cluster-setslot.md (`topics/cluster-spec.md:534`)
**Doc says:**
> Starting from Valkey 8.0, the `CLUSTER SETSLOT` command is replicated if the replicas are running Valkey version 8.0+. The primary node waits up to 2 seconds ...
**Problem:** `commands/cluster-setslot.md:86-96` has the same content (version, 2 s default, NOREPLICAS error, TIMEOUT syntax), and both copies have the units error in F1. Two copies drift.
**Evidence:** `commands/cluster-setslot.md:86-87`: "Starting from Valkey 8.0, `CLUSTER SETSLOT` is synchronously replicated to all healthy replicas running Valkey version 8.0+. By default, this synchronous replication must complete within 2 seconds."
**Fix:** Make `commands/cluster-setslot.md` canonical for the syntax, default, and error. In the spec, keep only the rationale paragraph (lines 548-555) and add "See [`CLUSTER SETSLOT`](../commands/cluster-setslot.md) for the `TIMEOUT` option."

### F10 [minor] AI-NATIVE: Tunables are described without config names or defaults (`topics/cluster-spec.md:868`)
**Doc says:**
> The replica replication link was disconnected from the primary for no longer than a given amount of time ... This time is user configurable.

Also line 804 ("N times the `NODE_TIMEOUT`") and line 1241 ("It sets a 60 second ban").
**Problem:** `NODE_TIMEOUT` appears 13 times but is never tied to `cluster-node-timeout`. The validity setting, the "N" multiplier, and the configurable ban are never named, so readers cannot quote or tune them.
**Evidence:** `valkey@9.1.2:src/config.c:3455` `cluster-node-timeout` default 15000. `:3391` `cluster-replica-validity-factor` default 10. `:3450` `cluster-blacklist-ttl` default 60. `src/cluster_legacy.h:10` `#define CLUSTER_FAIL_UNDO_TIME_MULT 2`.
**Fix:** At the first use of `NODE_TIMEOUT` (line 83), add: "`NODE_TIMEOUT` is the `cluster-node-timeout` config, default 15000 ms." Line 868: "...configurable with `cluster-replica-validity-factor` (default 10; the limit is `repl-ping-replica-period` + `cluster-node-timeout` × factor; 0 disables the check)." Line 804: "(2 × `NODE_TIMEOUT`)". Line 1241: "a ban of `cluster-blacklist-ttl` seconds (default 60)".

### F11 [minor] MISSING: Multi-database "additional restrictions" are never stated (`topics/cluster-spec.md:32`)
**Doc says:**
> Starting with version 9.0, Valkey cluster supports multiple databases, similar to standalone mode but with some additional restrictions.
**Problem:** The page names no restriction and links nowhere. No other topic page documents `cluster-databases`.
**Evidence:** `valkey@9.1.2:src/db.c:1847-1849`: `/* Not allowed in cluster mode ... */ addReplyError(c, "SWAPDB is not allowed in cluster mode");`. `valkey.conf:2128`: "dbid is a number between 0 and 'cluster-databases'-1." `cluster-databases` is `IMMUTABLE_CONFIG` (`src/config.c:3383`).
**Fix:** "...with these restrictions: `cluster-databases` can only be set at startup, `SWAPDB` is rejected, and every database shares the same slot ownership, so resharding moves a slot in all databases." Link to [`SELECT`](../commands/select.md).

### F12 [nit] STYLE: Non-inclusive "master" in prose (`topics/cluster-spec.md:906`)
**Doc says:**
> ### Masters reply to replica vote request

Also line 910 "Masters receive requests", lines 1071, 1072, 1090 "Master A ...", and line 1225 "Master nodes with non-empty data sets".
**Problem:** The rest of the page uses "primary". These are leftovers.
**Evidence:** Lines quoted above.
**Fix:** "### Primaries reply to replica vote request", "Primaries receive...", "Primary A...", "Primary nodes with non-empty data sets...".

### F13 [nit] OUTDATED: Sharded pub/sub introduced as a Redis OSS feature (`topics/cluster-spec.md:1256`)
**Doc says:**
> Redis OSS 7.0 and later features sharded pub/sub
**Problem:** Every Valkey release has sharded pub/sub. Naming Redis OSS here reads as if it is not a Valkey feature.
**Evidence:** Line quoted above. Valkey forked from Redis OSS 7.2.4.
**Fix:** "Valkey supports sharded pub/sub (`SSUBSCRIBE`/`SPUBLISH`), in which shard channels are assigned to slots..."

### F14 [nit] STYLE: Typos in SETSLOT replication section (`topics/cluster-spec.md:539`)
**Doc says:**
> If not all health replicas acknowledge

Line 548: "acknowledgement from health replicas". Line 552: "promoting `A` to relinquish its ownership".
**Problem:** "health" should be "healthy" and "promoting" should be "prompting".
**Evidence:** Lines quoted above.
**Fix:** "healthy replicas" (both lines), "prompting `A` to relinquish".

### F15 [nit] INACCURATE: Random ping is one node per second, not "a few" (`topics/cluster-spec.md:729`)
**Doc says:**
> Usually a node will ping a few random nodes every second
**Problem:** The cron pings one node per second: of 5 random nodes, the one with the oldest pong.
**Evidence:** `valkey@9.1.2:src/cluster_legacy.c:6213-6220`: "Ping some random node 1 time every 10 iterations ... `for (j = 0; j < 5; j++)`" (cron runs every 100 ms).
**Fix:** "Every second, a node picks 5 random nodes and pings the one it heard from least recently, so..."

### F16 [nit] STYLE: Fenced blocks without a language tag (`topics/cluster-spec.md:637`)
**Doc says:**
> ```
> 127.0.0.1:7000> cluster slots
**Problem:** The fences at lines 637, 968, 982, and 1010 have no language tag. Lines 151-152, 211-247, 287-290, 515, and 1264+ use indented code blocks and cannot take a tag.
**Evidence:** Lines quoted above.
**Fix:** Convert to fenced blocks with `text`, `ruby`, `c`, or `bash` as appropriate.

## Unverified
- topics/cluster-spec.md:557: "Starting from Valkey 8.0 ... elect a primary in empty shards." I did not trace the commit or test with an empty shard.
- topics/cluster-spec.md:197: "The pattern slot optimization is introduced in Valkey 8.0." I confirmed that `patternHashSlot` exists in 9.1.2 and is used by SCAN, KEYS, and SORT. I did not bisect the release.
