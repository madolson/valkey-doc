VERDICT: NEEDS-WORK
COUNTS: blocker=1 minor=9 nit=10

# Cluster tutorial (`cluster-tutorial`)

https://valkey.io/topics/cluster-tutorial/

The concepts hold up, but the hands-on half is Redis-era: the migration procedure assumes single-file AOF, the failover test fails on default config, sample output predates the current `CLUSTER NODES` format, and the 9.0 atomic slot migration path is absent. Fix the AOF migration steps first.

## Findings

### F1 [blocker] OUTDATED: Migration procedure assumes a single AOF file (`topics/cluster-tutorial.md:1044`)
**Doc says:**
> 2. Generate an append only file for all of your N primaries using the `BGREWRITEAOF` command, and waiting for the AOF file to be completely generated.
> 3. Save your AOF files from aof-1 to aof-N somewhere.
> 5. Stop all the cluster nodes, substitute their append only file with your pre-existing append only files, aof-1 for the first node, aof-2 for the second node, up to aof-N.

**Problem:** Since multi-part AOF, persistence is a directory of base, incr, and manifest files. There is no single "append only file" to save or substitute. A reader who copies one file gets a node that won't load it or loads the wrong data.
**Evidence:** 9.1.2 node after `BGREWRITEAOF`:
```
/tmp/docreview-cluster-tutorial/20261/appendonlydir:
appendonly.aof.2.base.rdb
appendonly.aof.2.incr.aof
appendonly.aof.manifest
```
`CONFIG GET appenddirname` returns `appendonlydir`.
**Fix:** Rewrite steps 2, 3, and 5 to work on the whole AOF directory: "Run `BGREWRITEAOF` on each of the N primaries and wait for `aof_rewrite_in_progress:0` in `INFO persistence`. Copy each node's entire `appenddirname` directory (default `appendonlydir`, including the manifest) to a safe location... Stop all cluster nodes and replace each node's `appendonlydir` with the copied directory from the matching old primary." You could also use RDB: `BGSAVE`, copy `dump.rdb`, then start the cluster nodes with `appendonly no` for the load.

### F2 [minor] INACCURATE: `cluster-require-full-coverage` stops all queries, not only writes (`topics/cluster-tutorial.md:215`)
**Doc says:**
> If this is set to yes, as it is by default, the cluster stops accepting writes if some percentage of the key space is not covered by any node.

**Problem:** With the default, nodes stop accepting all queries, reads included, as soon as one hash slot is uncovered. The page says writes only and implies a threshold.
**Evidence:** valkey@9.1.2:valkey.conf:1916-1917:
> By default cluster nodes stop accepting queries if they detect there is at least a hash slot uncovered (no available node is serving it).
**Fix:** "If set to yes (the default), the cluster stops accepting queries, reads and writes, when at least one hash slot is not served by any node. If set to no, nodes keep serving queries for the slots that are still covered."

### F3 [minor] INACCURATE: `cluster-allow-reads-when-down` said to keep writes available (`topics/cluster-tutorial.md:216`)
**Doc says:**
> It can also be used for when using Valkey Cluster with only one or two shards, as it allows the nodes to continue serving writes when a primary fails but automatic failover is impossible.

**Problem:** It is the opposite. The option keeps reads available, and writes still go down. This also contradicts the same bullet's earlier sentence.
**Evidence:** valkey@9.1.2:valkey.conf:1966-1968:
> A primary outage in a 1 or 2 shard configuration causes a read/write outage to the entire cluster without this option set, with it set there is only a write outage.
**Fix:** "...as it allows the nodes to continue serving reads when a primary fails and automatic failover is impossible. Writes to the failed primary's slots remain unavailable."

### F4 [minor] OUTDATED: Example app uses unmaintained, unlicensed redis-rb-cluster (`topics/cluster-tutorial.md:381`)
**Doc says:**
> #### Write an example app with redis-rb-cluster
> This section explains some basic usage of [redis-rb-cluster](https://github.com/antirez/redis-rb-cluster)

**Problem:** The repo was last pushed 2018-01-30 and has no license. Readers are told to install a Redis-branded Ruby client that no Valkey client page recommends. Lines 224, 393-398, 595, and 613 depend on it.
**Evidence:** `gh api repos/antirez/redis-rb-cluster --jq '.pushed_at, .license'` returns `2018-01-30T10:58:53Z` and an empty license.
**Fix:** Replace the example with a short cluster-aware loop written for a client listed under `clients/` (for example valkey-glide or valkey-py), or use `valkey-benchmark --cluster` as the background load. Rename the heading to "Run an example application against the cluster".
Tracked: valkey-io/valkey-doc#147

### F5 [minor] MISSING: Resharding section ignores atomic slot migration (`topics/cluster-tutorial.md:552`)
**Doc says:**
> After the final confirmation you'll see a message for every slot that valkey-cli is going to move from a node to another, and a dot will be printed for every actual key moved from one side to the other.

**Problem:** Valkey 9.0 added atomic slot migration, and valkey-cli 9.1.2 can use it for reshard and rebalance with `--cluster-use-atomic-slot-migration`. The page only covers the legacy key-by-key path and never links to the atomic slot migration page.
**Evidence:** valkey@9.1.2:src/valkey-cli.c:2800 `} else if (!strcmp(argv[i], "--cluster-use-atomic-slot-migration")) {` and :5383 `printf("Moving slot range %s from %s:%d to %s:%d via atomic slot migration", ...`. topics/atomic-slot-migration.md:7-10: "Valkey 9.0 introduced a option for migrating hash slots known as **atomic slot migration**".
**Fix:** After line 572, add: "Since Valkey 9.0, add `--cluster-use-atomic-slot-migration` to move slot ranges with `CLUSTER MIGRATESLOTS` instead of key-by-key `MIGRATE`. Both source and target must run 9.0 or later. See [Atomic slot migration](atomic-slot-migration.md)."

### F6 [minor] OUTDATED: Claims there's no automatic rebalancing (`topics/cluster-tutorial.md:574`)
**Doc says:**
> however currently there is no way for `valkey-cli` to automatically rebalance the cluster checking the distribution of keys across the cluster nodes and intelligently moving slots as needed. This feature will be added in the future.

**Problem:** `valkey-cli --cluster rebalance` exists. It balances slots by node weight, not by key count. "Will be added in the future" is stale filler. The aside at lines 523-525 ("but this is pretty trivial to implement") is the same kind of filler.
**Evidence:** valkey@9.1.2:src/valkey-cli.c:3749 `{"rebalance", clusterManagerCommandRebalance, -1,`
**Fix:** Replace lines 574-578 with: "To spread slots evenly across primaries, use `valkey-cli --cluster rebalance <host>:<port>`. Add `--cluster-use-empty-masters` to include primaries that have no slots. Rebalancing works on slot counts and optional `--cluster-weight`, not on key counts or memory." Cut "(but this is pretty trivial to implement)" at line 524.

### F7 [minor] OUTDATED: Redis-era env var name for `--cluster-yes` (`topics/cluster-tutorial.md:583`)
**Doc says:**
> Note that this option can also be activated by setting the `REDISCLI_CLUSTER_YES` environment variable.

**Problem:** The Valkey name is `VALKEYCLI_CLUSTER_YES`. The Redis name is only a fallback.
**Evidence:** valkey@9.1.2:src/valkey-cli.c:94-95 `#define CLI_CLUSTER_YES_ENV "VALKEYCLI_CLUSTER_YES"` / `#define OLD_CLI_CLUSTER_YES_ENV "REDISCLI_CLUSTER_YES"`. Lines 2936-2938 check the new name first.
**Fix:** "...by setting the `VALKEYCLI_CLUSTER_YES` environment variable. The legacy `REDISCLI_CLUSTER_YES` is still accepted."

### F8 [minor] EXAMPLE: `DEBUG SEGFAULT` fails with the tutorial's config (`topics/cluster-tutorial.md:672`)
**Doc says:**
> ```
> $ valkey-cli -p 7002 debug segfault
> Error: Server closed the connection
> ```

**Problem:** `DEBUG` is disabled by default, and the config at lines 243-249 doesn't enable it. The failover test step fails.
**Evidence:** 9.1.2 node with defaults:
```
ERR DEBUG command not allowed. If the enable-debug-command option is set to "local", you can run it from a local connection, otherwise you need to set this option in the configuration file, and then restart the server.
```
valkey@9.1.2:src/config.c:3370 `createEnumConfig("enable-debug-command", ..., PROTECTED_ACTION_ALLOWED_NO, ...)`
**Fix:** Crash the process without DEBUG: "Kill the primary process: `kill -9 $(valkey-cli -p 7002 info server | grep process_id | cut -d: -f2 | tr -d '\r')`". Or add `enable-debug-command local` to the test config at line 248 and say it is for testing only.

### F9 [minor] EXAMPLE: `CLUSTER NODES` output is pre-Valkey and line 870 is wrong (`topics/cluster-tutorial.md:870`)
**Doc says:**
> ```
> $ valkey-cli -p 7000 cluster nodes | grep slave | grep 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e
> f093c80dde814da99c5cf72a7dd01590792b783b 127.0.0.1:7006 replica 3c3a0c74... 
> ```
> and line 723: `* flags: master, replica, myself, fail, ...`

**Problem:** 9.1.2 prints the flag `slave`, not `replica`. As written, `grep slave` could never match the output shown. Every sample (lines 543, 663-665, 707-712, 811-817, 870-871) also uses the old format: no `@cport`, the `:0` self address, and epoch 0. Line 723 lists a `replica` flag that doesn't exist.
**Evidence:** 9.1.2 run:
```
a3c168963dc93138fac61dc94a49675d99a39c44 127.0.0.1:20260@30260 myself,master - 0 0 1 connected 0-5460
1dc9413fd27347b97f75d934c44e59241db35192 127.0.0.1:20266@30266 slave c960bb7c85e2c6bc01965c0189fa6ec32edeb65d 0 1792802720290 2 connected
```
**Fix:** Regenerate the samples from a 9.1.2 cluster on ports 7000-7006. Change `replica` to `slave` at lines 870-871. Change line 723 to "flags: `master`, `slave` (replica), `myself`, `fail`, ... (the `master`/`slave` flag names are kept for compatibility)". Change line 722 to "ip:port@cport". Link the token list to [CLUSTER NODES](../commands/cluster-nodes.md) instead of re-listing it.

### F10 [minor] OUTDATED: `del-node` works on a failed node (`topics/cluster-tutorial.md:894`)
**Doc says:**
> You should not use the `del-node` command because it tries to connect to all nodes and you will encounter a "connection refused" error.
> Instead, you can use the `call` command

**Problem:** On 9.1.2, `del-node` on an unreachable node prints a warning and still removes it from the cluster. The workaround is unnecessary.
**Evidence:** 9.1.2 run after shutting down node 20266:
```
>>> Removing node 1dc9413fd27347b97f75d934c44e59241db35192 from cluster 127.0.0.1:20260
>>> Sending CLUSTER FORGET messages to the cluster...
>>> WARNING: Could not connect to node 127.0.0.1:20266, unable to send CLUSTER RESET.
[OK] Node 1dc9413fd27347b97f75d934c44e59241db35192 removed from the cluster.
```
**Fix:** Replace lines 894-900 with: "`del-node` also works when the node being removed has failed. valkey-cli warns that it can't reset the node and still sends `CLUSTER FORGET` to the rest of the cluster. Don't restart the removed node with its old `nodes.conf`, or it will try to rejoin."

### F11 [nit] STYLE: Heading levels skip from h2 to h4 (`topics/cluster-tutorial.md:23`)
**Doc says:**
> ## Valkey Cluster 101
> #### Valkey Cluster TCP ports

**Problem:** Every subsection under an h2 is h4, and "Add a new node as a replica" is h5. The live page renders `<h2>` followed by `<h4>`. The pattern covers lines 23, 46, 80, 113, 134, 237, 294, 346, 381, 505, 585, 649, 731, 769, 833, 876, 902, 952, 1017.
**Evidence:** Live page: `<h2 id="valkey-cluster-101"` followed by `<h4 id="valkey-cluster-tcp-ports"`.
**Fix:** Promote every `####` to `###` and `#####` to `####`. Rename "Valkey Cluster 101" to "How Valkey Cluster shards and replicates data".

### F12 [nit] BROKEN: Backticks inside indented code render literally (`topics/cluster-tutorial.md:880`)
**Doc says:**
>     valkey-cli --cluster del-node 127.0.0.1:7000 `<node-id>`

**Problem:** Inside a code block the backticks are printed, so a copy-paste includes them. The same issue is at line 898.
**Evidence:** Live HTML: ``del-node 127.0.0.1:7000 `&lt;node-id&gt;` ``.
**Fix:** `valkey-cli --cluster del-node 127.0.0.1:7000 <node-id>` and `valkey-cli --cluster call 127.0.0.1:7000 cluster forget <node-id>`.

### F13 [nit] EXAMPLE: Prompt shows the Unix-socket form (`topics/cluster-tutorial.md:810`)
**Doc says:**
> valkey 127.0.0.1:7006> cluster nodes

**Problem:** valkey-cli only prefixes `valkey ` when connected over a Unix socket. A TCP connection shows `127.0.0.1:7006>`. The same issue is at line 862.
**Evidence:** valkey@9.1.2:src/valkey-cli.c:324-329: `if (config.ct == VALKEY_CONN_UNIX) { prompt = sdscatfmt(prompt, "valkey %s", ...` else `formatAddr(...)`.
**Fix:** Use `127.0.0.1:7006>` at lines 810 and 862.

### F14 [nit] EXAMPLE: Log samples don't match 9.1.2 format (`topics/cluster-tutorial.md:286`)
**Doc says:**
> [82462] 26 Nov 11:56:55.329 * No cluster configuration found, I'm 97a3...
> # Start of election delayed for 0 milliseconds (rank #0, offset 347540).
> # Starting a failover election for epoch 7545.

**Problem:** The line prefix format and several messages changed (lines 286, 750-755).
**Evidence:** 9.1.2 logs:
```
25335:M 24 Oct 2026 00:44:50.275 * No cluster configuration found, I'm a3c168963dc93138fac61dc94a49675d99a39c44
* Manual failover user request accepted (user request from 'id=6 addr=127.0.0.1:53808 ...').
* Start of election delayed for 0 milliseconds (rank #0, primary rank #0, offset 28).
* Starting a failover election for epoch 7, node config epoch is 1
```
**Fix:** Replace both samples with the 9.1.2 lines above, using `pid:role` prefixes and `*` level markers.

### F15 [nit] CONFUSING: Inconsistencies within the walkthrough (`topics/cluster-tutorial.md:943`)
**Doc says:**
> line 929: "the replica is killed at 4am, and the primary is killed at 6am"
> line 943: "when the primary will fail as well at 5am"
> line 443: "SET foo0 0"
> line 788: "start the server with `../valkey-server ./valkey.conf`"

**Problem:** The times disagree (6am vs 5am). The loop starts at `last+1`, so the first SET is `foo1`, which matches the output at line 489. Line 282 uses `valkey-server` and line 788 uses `../valkey-server`.
**Evidence:** Lines 428 `((last.to_i+1)..1000000000)`, 282 `valkey-server ./valkey.conf`.
**Fix:** Use "6am" at line 943. Change lines 443-445 to "SET foo1 1", "SET foo2 2", "SET foo3 3". Use `valkey-server ./valkey.conf` at line 788.

### F16 [nit] STYLE: Fenced blocks have no language tag (`topics/cluster-tutorial.md:243`)
**Doc says:**
> ```
> port 7000

**Problem:** Fences at lines 243, 266, 280, 353, 487, 541, 612, 631, 661, 671, 678, 705, 809, 868 have no language tag. Indented blocks at 57, 303, 518, 563, 572, 795, 838, 848, 862, 880, 898, 907 and 750 are also untagged.
**Evidence:** Source lines listed.
**Fix:** Tag config blocks `conf`, shell blocks `bash`, and output or log blocks `text`. Convert the indented blocks to fenced ones.

### F17 [nit] STYLE: "master" in prose and option names where Valkey aliases exist (`topics/cluster-tutorial.md:848`)
**Doc says:**
> --cluster-replica --cluster-master-id 3c3a0c74...
> line 724: "if it is a replica, the Node ID of the master"
> line 907: `CLUSTER REPLICATE <master-node-id>`

**Problem:** valkey-cli accepts `--cluster-primary-id`, so the old name isn't needed. Prose at lines 724 and 907 uses "master" where no literal requires it.
**Evidence:** valkey@9.1.2:src/valkey-cli.c:2740 `(!strcmp(argv[i], "--cluster-master-id") || !strcmp(argv[i], "--cluster-primary-id"))`.
**Fix:** Use `--cluster-primary-id` at line 848, "the Node ID of its primary" at line 724, and `CLUSTER REPLICATE <primary-node-id>` at line 907.

### F18 [nit] CONFUSING: Dangling master/slave note after the import section (`topics/cluster-tutorial.md:1060`)
**Doc says:**
> If not for backward compatibility, the Valkey project no longer uses the words "master" and "slave". Unfortunately in this command these words are part of the protocol

**Problem:** "This command" follows `--cluster import`, which has nothing to do with these words. The note belongs to the `CLUSTER NODES` flags.
**Evidence:** Placement at lines 1053-1060, while the `master`/`slave` tokens appear at lines 707-723.
**Fix:** Move the note under the `CLUSTER NODES` token list (line 729) as: "The `master` and `slave` flags are kept for client compatibility."

### F19 [nit] MISSING: `--cluster import` deletes source keys, and `--cluster-copy` isn't mentioned (`topics/cluster-tutorial.md:1056`)
**Doc says:**
> The command moves all the keys of a running instance (deleting the keys from the source instance) to the specified pre-existing Valkey Cluster.

**Problem:** Readers who want a non-destructive import aren't told that the option exists.
**Evidence:** valkey@9.1.2:src/valkey-cli.c:2788 `} else if (!strcmp(argv[i], "--cluster-copy")) {`
**Fix:** Append: "Add `--cluster-copy` to keep the keys on the source and `--cluster-replace` to overwrite existing keys in the cluster. Usage: `valkey-cli --cluster import <cluster-host>:<port> --cluster-from <source-host>:<port> [--cluster-copy] [--cluster-replace]`."

### F20 [nit] AI-NATIVE: Intro contradicts itself, cluster bus wording is loose (`topics/cluster-tutorial.md:7`)
**Doc says:**
> line 7: "This topic will teach you how to set up, test, and operate Valkey Cluster in production."
> line 10: "If you plan to run a production Valkey Cluster deployment ... consult the Valkey Cluster specification"
> line 25: "Every Valkey Cluster node requires two open TCP connections"
> line 41: "The Valkey cluster bus protocol (running on client port + 10000) does not perform application-layer authentication of messages by default."

**Problem:** Line 10 sends production readers away from a page that line 7 says is for production. Line 25 means ports, not connections. Line 41 ignores `cluster-port` and "by default" implies an auth option that doesn't exist. The front matter description, "Horizontal scaling with Valkey Cluster", doesn't say this is a hands-on guide.
**Evidence:** Lines quoted. No cluster bus authentication config exists in valkey@9.1.2:src/config.c.
**Fix:** Line 10: "For protocol internals (gossip, failure detection, epochs), see the [Valkey Cluster specification](cluster-spec.md)." Line 25: "requires two open TCP ports". Line 41: "(the client port + 10000, or `cluster-port` if set) does not authenticate messages." Description: "Set up, reshard, fail over, scale, and upgrade a Valkey Cluster with valkey-cli."
