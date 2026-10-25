VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=5

# Latency monitoring (`latency-monitor`)

https://valkey.io/topics/latency-monitor/

Core facts (threshold config, default 0, 160-sample series, same-second merge by max) are correct for 9.1.2. The event list is the main problem: it names one event that no longer exists and omits about a dozen that 9.1.2 emits, so readers cannot interpret `LATENCY LATEST` output.

## Findings

### F1 [minor] OUTDATED: `aof-rewrite-diff-write` event no longer exists (`topics/latency-monitor.md:70`)
**Doc says:**
> * `aof-rewrite-diff-write`: writing the differences accumulated while performing `BGREWRITEAOF`.
**Problem:** No code path in 9.1.2 records this event. The rewrite diff buffer was removed with multi-part AOF. The name survives only as a string match in the `LATENCY DOCTOR` advice table.
**Evidence:** `git grep -n aof-rewrite-diff-write 9.1.2 -- src/*.c` returns only:
```
9.1.2:src/latency.c:326:        if (!strcasecmp(event, "aof-rewrite-diff-write") || !strcasecmp(event, "aof-rename")) {
```
No `latencyAddSampleIfNeeded("aof-rewrite-diff-write", ...)` call exists.
**Fix:** Delete line 70.

### F2 [minor] MISSING: event list omits events that 9.1.2 records (`topics/latency-monitor.md:57`)
**Doc says:**
> The framework monitors and logs latency spikes in the execution time of these events:
**Problem:** The list presents itself as complete but omits events a reader will see in `LATENCY LATEST`.
**Evidence:** `latencyAddSampleIfNeeded` call sites in 9.1.2 not on the page:
```
src/db.c:1957:            "expire-del"
src/evict.c:629:          "eviction-lazyfree"
src/blocked.c:151:        "command-unblocking"
src/server.c:1774:        "while-blocked-cron"
src/server.c:2029:        "module-acquire-GIL"
src/cluster_legacy.c:1084-1144: "cluster-config-open", "cluster-config-write", "cluster-config-fsync",
                          "cluster-config-rename", "cluster-config-dir-fsync", "cluster-config-close", "cluster-config-unlink"
src/module.c:8237:        VM_LatencyAddSample(event, ...)   (module-defined event names)
```
**Fix:** Add:
* `expire-del`: deleting a single expired key.
* `eviction-lazyfree`: waiting for lazy-free memory release during eviction.
* `command-unblocking`: replying to a client unblocked from a blocking command.
* `while-blocked-cron`: periodic work done while the server is busy loading or running a long script.
* `module-acquire-GIL`: acquiring the module global lock after the event loop sleeps.
* `cluster-config-open`, `cluster-config-write`, `cluster-config-fsync`, `cluster-config-rename`, `cluster-config-dir-fsync`, `cluster-config-close`, `cluster-config-unlink`: steps of saving `nodes.conf` in cluster mode.
Then add: "Modules can record their own events with `ValkeyModule_LatencyAddSample`."

### F3 [minor] MISSING: subcommand list omits HISTOGRAM and HELP, and has no links (`topics/latency-monitor.md:94`)
**Doc says:**
> * `LATENCY LATEST` - returns the latest latency samples for all events.
> ...
> Refer to each subcommand's documentation page for further information.
**Problem:** `LATENCY HISTOGRAM` and `LATENCY HELP` exist in 9.1.2 and have pages (`commands/latency-histogram.md`, `commands/latency-help.md`). Line 100 points readers at pages without linking any of them.
**Evidence:** 9.1.2 `LATENCY HELP`:
```
HISTOGRAM [COMMAND ...]
    Return a cumulative distribution of latencies in the format of a histogram for the specified command names.
HELP
    Print this help.
```
**Fix:** Link each bullet to its page (`../commands/latency-latest.md`, etc.) and add:
* [`LATENCY HISTOGRAM`](../commands/latency-histogram.md) - returns per-command latency histograms. Unlike the other subcommands, it does not depend on `latency-monitor-threshold`.
* [`LATENCY HELP`](../commands/latency-help.md) - lists the subcommands.
Tracked: valkey-io/valkey-doc#489

### F4 [minor] OUTDATED: "Valkey is single threaded" (`topics/latency-monitor.md:16`)
**Doc says:**
> Finally, Valkey is single threaded.
**Problem:** Valkey has I/O threads (`io-threads`) and background threads for lazy free and fsync. Command execution is single threaded, which is the point this paragraph makes.
**Evidence:** `valkey.conf` in 9.1.2 documents `io-threads`. `topics/benchmark.md` describes Valkey as mostly single threaded.
**Fix:** "Finally, Valkey executes commands on a single main thread."
Tracked: valkey-io/valkey-doc#112, valkey-io/valkey-doc#489

### F5 [nit] INACCURATE: all-time maximum is per event, not per element (`topics/latency-monitor.md:55`)
**Doc says:**
> * Records the all-time maximum latency for every element.
**Problem:** Each time series keeps one `max` for the event. Elements are individual samples. The bullet also has no subject.
**Evidence:** `valkey@9.1.2:src/latency.h:53`: `uint32_t max; /* Max latency observed for this event. */`
**Fix:** "* Each time series also records the all-time maximum latency for its event, since the last reset."

### F6 [nit] INACCURATE: `fast-command` is defined by the command's FAST flag, not its complexity (`topics/latency-monitor.md:42`)
**Doc says:**
> while `fast-command` is the event name for the monitoring of the O(1) and O(log N) commands.
**Problem:** Commands go to `fast-command` when they carry the `CMD_FAST` flag. Complexity is only a rough guide.
**Evidence:** `valkey@9.1.2:src/server.c:3979`: `char *latency_event = (real_cmd->flags & CMD_FAST) ? "fast-command" : "command";`
**Fix:** Lines 42 and 60: "`fast-command`: commands flagged `fast` (shown in `COMMAND INFO` and the `@fast` ACL category), usually O(1) or O(log N)." Line 59: "`command`: all other commands."

### F7 [nit] STYLE: typo "allways" (`topics/latency-monitor.md:63`)
**Doc says:**
> invoked by the `appendfsync allways` policy.
**Problem:** The config value is `always`. A reader copying it gets an invalid value.
**Fix:** `appendfsync always`.
Tracked: valkey-io/valkey-doc#489

### F8 [nit] EXAMPLE: code block has no language tag (`topics/latency-monitor.md:85`)
**Doc says:**
>     CONFIG SET latency-monitor-threshold 100
**Problem:** Indented block. The live page renders it as `data-lang="plain"`.
**Fix:** Use a fenced block tagged `bash` with `valkey-cli CONFIG SET latency-monitor-threshold 100`, and add one sentence saying to persist it in `valkey.conf` or with `CONFIG REWRITE`.

### F9 [nit] AI-NATIVE: intro is generic and does not say what the page covers (`topics/latency-monitor.md:6`)
**Doc says:**
> Valkey is often used for demanding use cases, where it serves a large number of queries per second per instance...
**Problem:** Lines 6 to 22 are generic background. The page's subject, the threshold config, and its default only appear at line 85.
**Fix:** Replace lines 6 to 22 with: "The latency monitor records operations that block the Valkey main thread for longer than `latency-monitor-threshold` milliseconds (default 0, disabled). This page lists the monitored events, explains how samples are stored, and shows how to enable the monitor and read its data with `LATENCY`." Keep the link to latency.md.
