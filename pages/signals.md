VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=5

# Signal handling (`signals`)

https://valkey.io/topics/signals/

The page is mostly accurate for 9.1.2 but incomplete. The fix that matters most: document `shutdown-on-sigterm` / `shutdown-on-sigint` and the second-SIGINT immediate exit, because both change what SIGTERM and SIGINT actually do.

## Findings

### F1 [minor] MISSING: SIGTERM/SIGINT behavior is configurable, and a second SIGINT exits immediately (`topics/signals.md:10`)
**Doc says:**
> The `SIGTERM` and `SIGINT` signals tell Valkey to shut down gracefully. ... it schedules a shutdown similar to the one performed by the `SHUTDOWN` command.

**Problem:** The page leaves out `shutdown-on-sigterm` and `shutdown-on-sigint`. These configs control which `SHUTDOWN` flags a signal applies, so an operator can make SIGTERM skip the save or skip the replica wait. The page also leaves out what happens on a second SIGINT during a pending shutdown: the server exits at once with code 1 and skips the save and the replica wait. A reader who presses Ctrl+C twice loses unsaved data and gets no warning from this page.
**Evidence:** valkey@9.1.2:src/config.c:104-112, 3375-3376
```c
configEnum shutdown_on_sig_enum[] = {
    {"default", SHUTDOWN_NOFLAGS}, {"save", SHUTDOWN_SAVE}, {"nosave", SHUTDOWN_NOSAVE},
    {"now", SHUTDOWN_NOW}, {"force", SHUTDOWN_FORCE}, {"safe", SHUTDOWN_SAFE},
    {"failover", SHUTDOWN_FAILOVER}, {NULL, 0}};
createEnumConfig("shutdown-on-sigint", ...
createEnumConfig("shutdown-on-sigterm", ...
```
valkey@9.1.2:src/server.c:7001-7012
```c
if (server.shutdown_asap && sig == SIGINT) {
    serverLogRawFromHandler(LL_WARNING, "You insist... exiting now.");
    ...
    exit(1); /* Exit with an error since this was not a clean shutdown. */
```
**Fix:** After line 14, add: "The flags the signal applies are set by `shutdown-on-sigterm` and `shutdown-on-sigint` (default `default`). They accept any combination of `save`, `nosave`, `now`, `force`, `safe`, and `failover`, with the same meaning as the matching [`SHUTDOWN`](../commands/shutdown.md) options. For example, `shutdown-on-sigterm nosave now` makes SIGTERM skip the final save and the replica wait. If a second `SIGINT` arrives while a shutdown is pending, the server logs `You insist... exiting now.` and exits immediately with exit code 1. It does not save and does not wait for replicas."

### F2 [minor] DUPLICATE: Shutdown sequence and data-loss paragraph copied from SHUTDOWN (`topics/signals.md:20`)
**Doc says:**
> Since Redis OSS 7.0, the server waits for lagging replicas up to a configurable `shutdown-timeout`, 10 seconds by default, before shutting down. This provides a best effort to minimize the risk of data loss ... trigger a manual `FAILOVER` (or `CLUSTER FAILOVER`) ...

**Problem:** Lines 36-39 are a near-verbatim copy of commands/shutdown.md:69-72 ("The server waits for lagging replicas up to a configurable `shutdown-timeout`, by default 10 seconds ... trigger a manual [`FAILOVER`](failover.md) (or [`CLUSTER FAILOVER`](cluster-failover.md))"). The step list at lines 22-24 repeats commands/shutdown.md:3-5. Two copies of the same sequence will drift apart. They already differ: this page lists PID and socket removal, and SHUTDOWN does not.
**Evidence:** commands/shutdown.md:3-5 and :69-72 (quoted above). commands/shutdown.md:19-20 already says signals run "the same shutdown sequence" and links here.
**Fix:** Make commands/shutdown.md canonical for the shutdown sequence and the data-loss advice. On this page, delete lines 36-39 and replace them with "To reduce data loss when shutting down a primary, see [Minimize the risk of data loss](../commands/shutdown.md#minimize-the-risk-of-data-loss)." Keep only the signal-specific parts here: scheduling, the 0.1 s delay, Lua, F1, and the failure cases at lines 32-34.

### F3 [minor] INACCURATE: Error is `MISCONF`, not `MISCONFIG`, and it depends on a config (`topics/signals.md:73`)
**Doc says:**
> Valkey will reply to all write commands with a `MISCONFIG` error.

**Problem:** The error prefix is `MISCONF`. The write block also applies only when `stop-writes-on-bgsave-error` is `yes` (the default), and the page never names that config, so a reader has no way to turn the behavior off.
**Evidence:** 9.1.2 run. BGSAVE into an unwritable dir, then a write:
```
127.0.0.1:21400> set a c
MISCONF Valkey is configured to save RDB snapshots, but it's currently unable to persist to disk. Commands that may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Valkey logs for details about the RDB error.
```
valkey@9.1.2:src/server.c:2080 `"-MISCONF %s is configured to save RDB snapshots, ..."`
**Fix:** Change line 73 to "Valkey replies to all write commands with a `MISCONF` error." Change lines 68-70 to "...Valkey stops accepting write commands if `stop-writes-on-bgsave-error` is `yes` (the default):"

### F4 [minor] MISSING: SIGABRT is handled as a crash, and two configs control the crash handler (`topics/signals.md:41`)
**Doc says:**
> The following signals are handled as a Valkey crash: SIGSEGV, SIGBUS, SIGFPE, SIGILL
> A fast memory test is performed as a first check of the reliability of the crashing system.

**Problem:** SIGABRT also gets the crash handler, but the page does not list it. The crash report applies only if `crash-log-enabled` is `yes`, and the memory test runs only if `crash-memcheck-enabled` is `yes`. Both default to `yes`, and the page names neither.
**Evidence:** valkey@9.1.2:src/debug.c:2378-2384
```c
if (server.crashlog_enabled) {
    sigaction(SIGSEGV, &act, NULL);
    sigaction(SIGBUS, &act, NULL);
    sigaction(SIGFPE, &act, NULL);
    sigaction(SIGILL, &act, NULL);
    sigaction(SIGABRT, &act, NULL);
```
valkey@9.1.2:src/debug.c:2208 `if (server.memcheck_enabled) {`. valkey@9.1.2:src/config.c:3306-3307 `crash-log-enabled` and `crash-memcheck-enabled`, both default 1.
**Fix:** Change the heading to "SIGSEGV, SIGBUS, SIGFPE, SIGILL and SIGABRT" and add `* SIGABRT` to the list. Change line 53 to "If `crash-memcheck-enabled` is `yes` (default), a fast memory test is performed...". After the list, add: "Setting `crash-log-enabled no` disables this handler. The signal then takes its default action with no bug report."

### F5 [nit] INACCURATE: PID file removal is not tied only to daemonize (`topics/signals.md:28`)
**Doc says:**
> If the server is daemonized, the PID file is removed.

**Problem:** On a graceful shutdown the PID file is removed whenever `daemonize` is on or `pidfile` is set. A non-daemonized server with a `pidfile`, which is common under systemd, also removes it.
**Evidence:** valkey@9.1.2:src/server.c (finishShutdown) `if (server.daemonize || server.pidfile) { serverLog(LL_NOTICE, "Removing the pid file."); unlink(server.pidfile); }`
**Fix:** "If the server is daemonized or `pidfile` is set, the PID file is removed." Line 54 is correct as written. The crash path checks `server.daemonize && server.supervised == 0 && server.pidfile`.

### F6 [nit] MISSING: SIGUSR1 also works on the AOF rewrite child, and `BGSAVE CANCEL` exists (`topics/signals.md:77`)
**Doc says:**
> ## Kill the RDB file without errors
> ... This can be done using the signal `SIGUSR1`.

**Problem:** The heading says "RDB file", but the signal kills the child process, not a file. SIGUSR1 is also exempt from error handling for the AOF rewrite child. Since 8.1.0, `BGSAVE CANCEL` does the same thing without sending a signal by hand.
**Evidence:** valkey@9.1.2:src/aof.c:2905 `if (bysignal != SIGUSR1) {`. valkey@9.1.2:src/rdb.c:3699 `if (bysignal != SIGUSR1) server.lastbgsave_status = C_ERR;`. valkey@9.1.2:src/commands/bgsave.json history `"8.1.0", "Added the \`CANCEL\` option."`
**Fix:** Rename the heading to "Killing a save or rewrite child without errors". Add: "This applies to both the RDB-saving child and the AOF rewrite child. From Valkey 8.1, [`BGSAVE CANCEL`](../commands/bgsave.md) cancels a background save without signaling the child directly."

### F7 [nit] STYLE: Commands and configs are never linked (`topics/signals.md:12`)
**Doc says:**
> similar to the one performed by the `SHUTDOWN` command ... `SCRIPT KILL` ... `CLIENT PAUSE` ... `FAILOVER` (or `CLUSTER FAILOVER`)

**Problem:** The page has no links at all. `SHUTDOWN` (line 12), `SCRIPT KILL` (17), `CLIENT PAUSE` (23), and `FAILOVER`/`CLUSTER FAILOVER` (39) are plain code spans, so readers cannot reach the flags the page refers to. Line 17 also leaves out `FUNCTION KILL` for long-running functions. The BUSY error itself names it: `-BUSY ... You can only call FUNCTION KILL or SHUTDOWN NOSAVE.` (valkey@9.1.2:src/server.c:2076).
**Fix:** Link each one, for example [`SHUTDOWN`](../commands/shutdown.md), [`SCRIPT KILL`](../commands/script-kill.md), [`CLIENT PAUSE`](../commands/client-pause.md). Change line 16-17 to "If the server is blocked by a long-running Lua script or function, kill it with [`SCRIPT KILL`](../commands/script-kill.md) or [`FUNCTION KILL`](../commands/function-kill.md) if possible."

### F8 [nit] CONFUSING: "0.1 seconds" assumes the default `hz` (`topics/signals.md:14`)
**Doc says:**
> with a possible additional delay of 0.1 seconds or less.

**Problem:** `serverCron` checks for the pending shutdown, so the delay is one cron period (1/`hz` seconds). It is 0.1 s only at the default `hz 10`.
**Evidence:** valkey@9.1.2:src/server.c:1539-1547, inside `serverCron`: `if (server.shutdown_asap && !isShutdownInitiated()) { ... if (prepareForShutdown(NULL, shutdownFlags) == C_OK) exit(0);`
**Fix:** "with a possible additional delay of up to one server cron cycle (0.1 seconds at the default `hz 10`)."

### F9 [nit] STYLE: Typo "IF" (`topics/signals.md:32`)
**Doc says:**
> IF the RDB file can't be saved, the shutdown fails

**Problem:** Wrong capitalization.
**Fix:** "If the RDB file can't be saved, the shutdown fails (unless the `force` flag is set through `shutdown-on-sigterm`/`shutdown-on-sigint`), and the server continues to run..." The `force` exception is from valkey@9.1.2:src/server.c finishShutdown: `if (force) { serverLog(LL_WARNING, "Error trying to save the DB. Exit anyway.");`.
