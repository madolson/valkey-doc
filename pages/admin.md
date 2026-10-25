VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=8 nit=4

# Administration (`admin`)

https://valkey.io/topics/admin/

The page is Redis-era advice that ignores defaults Valkey now ships: diskless sync, per-process THP disable, protected mode, and the `FAILOVER` command. The most important fix is the no-downtime upgrade procedure. It hand-rolls a racy switchover that `FAILOVER` (since 6.2.0) already does safely. A general rewrite is open in valkey-io/valkey-doc#473 (fixes #297), but it does not address the specific findings below.

## Findings

### F1 [minor] OUTDATED: Diskless replication is already the default (`topics/admin.md:36`)
**Doc says:**
> If you use replication, Valkey performs RDB saves even if persistence is disabled. (This does not apply to diskless replication.) If you don't have disk usage on the primary, enable diskless replication.
**Problem:** `repl-diskless-sync` defaults to `yes`, so a default primary does not write an RDB to disk for full syncs. The advice tells readers to enable something that is already on, and it implies the reverse is the default.
**Evidence:** valkey@9.1.2:src/config.c:3285 `createBoolConfig("repl-diskless-sync", ..., server.repl_diskless_sync, 1, ...)`. valkey@9.1.2:valkey.conf:759 `repl-diskless-sync yes`. Live 9.1.2 run: `CONFIG GET repl-diskless-sync` returned `yes`.
**Fix:** "Full synchronization uses diskless replication by default (`repl-diskless-sync yes`): the primary streams the RDB to replicas over the socket and does not write it to disk. If you set `repl-diskless-sync no`, the primary writes an RDB file for each full sync even when persistence is disabled."

### F2 [minor] INACCURATE: Security default omits protected mode and the shipped bind (`topics/admin.md:42`)
**Doc says:**
> By default, Valkey does not require any authentication and listens to all the network interfaces. This is a big security issue if you leave Valkey exposed on the internet
**Problem:** This is only half true. With no config file, the server binds `* -::*`, but `protected-mode yes` rejects non-loopback clients when no password is set. The shipped `valkey.conf` binds to localhost only. Readers get a wrong picture of the default exposure, and they never learn that protected mode exists or that disabling it is the dangerous step.
**Evidence:** valkey@9.1.2:src/server.h:159 `#define CONFIG_DEFAULT_BINDADDR {"*", "-::*"}`. valkey@9.1.2:src/config.c:3272 `createBoolConfig("protected-mode", ..., 1, ...)`. valkey@9.1.2:valkey.conf:88 `bind 127.0.0.1 -::1`, :112 `protected-mode yes`.
**Fix:** "By default Valkey requires no authentication. Without a config file it listens on all interfaces, but protected mode (`protected-mode yes`) refuses connections from non-loopback addresses until you set a password or an explicit `bind`. The shipped `valkey.conf` binds to `127.0.0.1 -::1`. Before you expose Valkey on a network, configure [ACLs](acl.md) or `requirepass`, and see the [security page](security.md)."

### F3 [minor] OUTDATED: THP is disabled per process by default (`topics/admin.md:16`)
**Doc says:**
> To ensure the Linux kernel feature Transparent Huge Pages does not impact Valkey memory usage and latency, run the command: `echo never > /sys/kernel/mm/transparent_hugepage/enabled` to disable it.
**Problem:** `disable-thp yes` is the default. On Linux 3.5+ the server calls `prctl(PR_SET_THP_DISABLE)` for its own process when the kernel THP setting is `always`. The system-wide `echo` is a fallback for when that fails, which the server reports with a startup warning. As written, the page makes a host-wide kernel change mandatory, and the command does not persist across reboots.
**Evidence:** valkey@9.1.2:src/config.c:3309 `createBoolConfig("disable-thp", ..., server.disable_thp, 1, ...)`. valkey@9.1.2:src/server.c:6855-6860 `if (checkTHPEnabled(&err_msg) < 0) { ... if (THPDisable() == 0) { server.thp_enabled = 0; } else { serverLog(LL_WARNING, ...` and :6843 `prctl(PR_SET_THP_DISABLE, 1, 0, 0, 0)`. valkey@9.1.2:valkey.conf:1549-1553 "On systems in which it is set to "always", the server will attempt to disable it specifically for the server process".
**Fix:** "Valkey disables Transparent Huge Pages for its own process by default (`disable-thp yes`). If the startup log shows a THP warning, the per-process disable failed. In that case run `echo never > /sys/kernel/mm/transparent_hugepage/enabled` and persist it at boot."

### F4 [minor] MISSING: Upgrade procedure ignores `FAILOVER` and has a write-loss race (`topics/admin.md:56-70`)
**Doc says:**
> Allow writes to the replica using `CONFIG SET replica-read-only no`. ... Configure all your clients to use the new instance (the replica). Note that you may want to use the `CLIENT PAUSE` command ... elect the replica to primary using the `REPLICAOF NO ONE` command
**Problem:** Clients write to a node that is still a replica. If the link to the primary drops and a full resync happens, those writes are lost, and writes on the replica are never sent back to the primary. `CLIENT PAUSE` is "may want", not a required step. `FAILOVER` (since 6.2.0) pauses writes on the primary, waits for the replica to catch up, and promotes it in one command. The page never mentions it.
**Evidence:** valkey@9.1.2:src/commands/failover.json:6 `"since": "6.2.0"`. The command is documented at commands/failover.md.
**Fix:** Replace the last three bullets with: "On the old primary, run `FAILOVER TO <replica-host> <replica-port>`. The primary pauses writes, waits for the replica to catch up, promotes it, and becomes its replica. Point clients at the new primary, then shut down the old one. See [`FAILOVER`](../commands/failover.md)." Drop the `replica-read-only no` step. If you keep the manual path, make `CLIENT PAUSE <timeout> WRITE` a required step before `REPLICAOF NO ONE`.

### F5 [minor] CONFUSING: Sentinel/Cluster upgrade sentence is wrong at the end (`topics/admin.md:72`)
**Doc says:**
> the simplest way to upgrade to newer versions is to upgrade one replica after the other. Then you can perform a manual failover to promote one of the upgraded replicas to primary, and finally promote the last replica.
**Problem:** "Promote the last replica" makes no sense. The last step is upgrading the old primary, which is now a replica. The sentence also does not name the command to run for each mode.
**Fix:** "...upgrade the replicas one at a time. Then trigger a manual failover to promote an upgraded replica: `SENTINEL FAILOVER <primary-name>` with Sentinel, or `CLUSTER FAILOVER` on the replica with Cluster. Finally, upgrade the old primary, which is now a replica."

### F6 [minor] OUTDATED: EC2 section is obsolete and duplicated (`topics/admin.md:44-48`)
**Doc says:**
> Use HVM based instances, not PV based instances.
**Problem:** Current-generation EC2 instances are all HVM (Nitro), so PV advice does not apply to anything a reader can launch today. The same advice appears in topics/latency.md:23 ("make sure you use HVM based modern EC2 instances, like m3.medium"). The other bullets are generic: "handled with care" and "you may want to try diskless replication", which is already the default (F1).
**Fix:** Delete the "Running Valkey on EC2" section. If you keep an EBS note, merge it into Memory/persistence as: "On network block storage such as EBS, fsync latency can stall AOF writes. Consider `appendfsync everysec` and monitor `latency` events."

### F7 [minor] DUPLICATE: Persistence-off primary restart covered in depth by replication.md (`topics/admin.md:38`)
**Doc says:**
> ensure that either your primary has persistence enabled, or that it does not automatically restart on crashes. Replicas will try to maintain an exact copy of the primary, so if a primary restarts with an empty data set, replicas will be wiped as well.
**Problem:** topics/replication.md:42 ("Safety of replication when primary has persistence turned off") explains the same failure mode with a worked scenario. replication.md should be canonical.
**Fix:** Keep one line and link to it: "If the primary runs without persistence, disable auto-restart. See [Safety of replication when primary has persistence turned off](replication.md#safety-of-replication-when-primary-has-persistence-turned-off)."

### F8 [minor] CONFUSING: Swap advice is unexplained and contradicts itself (`topics/admin.md:20`)
**Doc says:**
> Ensured that swap is enabled and that your swap file size is equal to amount of memory on your system. ... When swapping is enabled, you can detect latency spikes and act on them.
**Problem:** The bullet tells readers to size swap equal to RAM, but it never explains that swapping a live Valkey causes severe latency. topics/latency.md treats swapping as a latency cause to eliminate. "Detect latency spikes and act on them" does not say how. Readers can come away thinking swap is a performance feature.
**Fix:** "Enable swap as a safety net, not as extra capacity. Without swap, a Valkey process that exceeds available memory is killed by the OOM killer. With swap it slows down sharply instead, which you can see as latency spikes (see [latency diagnosis](latency.md)) and fix before data is lost. Set `maxmemory` so that normal operation never swaps."

### F9 [nit] AI-NATIVE: No intro, and the "Imaging" heading carries no meaning (`topics/admin.md:6`, `topics/admin.md:28`)
**Doc says:**
> ## Valkey setup tips
> ### Imaging
> * When running under daemontools, use `daemonize no`.
**Problem:** The page opens straight into a heading, with nothing saying who it is for. "Imaging" does not describe the section. daemontools is rare. systemd and the `supervised` option (default `no`) are missing.
**Fix:** Add an opening line: "Production checklist for operators running self-managed Valkey: OS tuning, memory sizing, replication, security, and zero-downtime upgrades." Rename "Imaging" to "Running under a process supervisor" and write: "Under systemd, upstart, or daemontools, set `daemonize no`. With systemd, also set `supervised systemd` so Valkey signals readiness."

### F10 [nit] AI-NATIVE: "Non-trivial replication backlog" has no number (`topics/admin.md:34`)
**Doc says:**
> Set up a non-trivial replication backlog in proportion to the amount of memory Valkey is using.
**Problem:** The bullet gives neither the config name nor the default, so readers cannot act on it.
**Evidence:** valkey@9.1.2:src/config.c:3463 `"repl-backlog-size", ... 10 * 1024 * 1024 ... /* Default: 10mb */`. Live 9.1.2 run: `CONFIG GET repl-backlog-size` returned `10485760`.
**Fix:** "Size `repl-backlog-size` (default 10mb) to hold the writes produced during the longest replica disconnect you expect to tolerate. Replicas that reconnect within the backlog do a partial resync instead of a full sync."

### F11 [nit] STYLE: Typos and grammar (`topics/admin.md:11`, `:20`, `:22`)
**Doc says:**
> on other OpenBSD, NetBSD ... / Ensured that swap is enabled ... equal to amount of memory / when the system memory limit is near to be reached
**Fix:** "on OpenBSD, NetBSD, ..." / "Ensure that swap is enabled ... equal to the amount of memory" / "when the system memory limit is nearly reached".

### F12 [nit] STYLE: `CONFIG GET *` pointer is vague (`topics/admin.md:52`)
**Doc says:**
> You can also switch from AOF to RDB snapshots persistence, or the other way around, without restarting Valkey. Check the output of the `CONFIG GET *` command for more information.
**Problem:** `CONFIG GET *` lists values. It does not explain how to switch persistence modes.
**Fix:** Replace the last sentence with "See [Persistence](persistence.md) for how to switch between RDB and AOF at runtime."
