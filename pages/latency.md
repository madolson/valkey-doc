VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=5

# Diagnosing latency issues (`latency`)

https://valkey.io/topics/latency/

Mostly accurate Redis-era content with a few stale or wrong mechanics. The most important fix is the expire section, which describes a superseded algorithm (25% threshold, unbounded blocking) that contradicts commands/expire.md and the 9.1.2 source.

## Findings

### F1 [minor] INACCURATE: Active expire algorithm is outdated and duplicates commands/expire.md (`topics/latency.md:537`)
**Doc says:**
> Sample `ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP` keys, evicting all the keys already expired.
> If the more than 25% of the keys were found expired, repeat.

> Valkey can block in order to get the percentage of keys already expired below 25%.

**Problem:** The constant is `ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP`, the repeat threshold is 10% (minus effort), and the slow cycle is time-bounded to 25% of the hz period, so it cannot block until the stale ratio drops. A fast cycle in `beforeSleep` and the `active-expire-effort` config (lines 527-548 mention neither) also exist. commands/expire.md:140-169 already documents the current algorithm correctly.
**Evidence:** valkey@9.1.2:src/expire.c:122-125
```
#define ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP 20    /* Keys for each DB loop. */
#define ACTIVE_EXPIRE_CYCLE_FAST_DURATION 1000  /* Microseconds. */
#define ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC 25   /* Max % of CPU to use. */
#define ACTIVE_EXPIRE_CYCLE_ACCEPTABLE_STALE 10 /* % of stale keys after which */
```
valkey@9.1.2:src/expire.c:202: `config_cycle_acceptable_stale = ACTIVE_EXPIRE_CYCLE_ACCEPTABLE_STALE - activeExpireEffort();`
valkey@9.1.2:src/expire.c:361: `: (data.sampled == 0 || (data.expired * 100 / data.sampled) > config_cycle_acceptable_stale);`
**Fix:** Replace lines 530-546 with: "Valkey reclaims expired keys lazily on access and actively in a background cycle. The active cycle repeats while more than 10% of sampled keys are expired, but is capped at 25% of CPU time per cycle (default `active-expire-effort 1`). If many keys expire in the same second, for example from heavy `EXPIREAT` use with one timestamp, the cycle runs up to that cap and adds latency to every tick. Raising `active-expire-effort` reclaims memory faster at the cost of more CPU and latency. See [How Valkey reclaims expired keys](../commands/expire.md#how-valkey-reclaims-expired-keys) for the algorithm." Keep commands/expire.md canonical.

### F2 [minor] OUTDATED: THP advice ignores `disable-thp`, which handles this by default (`topics/latency.md:24`)
**Doc says:**
> Transparent huge pages must be disabled from your kernel. Use `echo never > /sys/kernel/mm/transparent_hugepage/enabled` to disable them, and restart your Valkey process.

Also line 248-250: "Make sure to **disable transparent huge pages** using the following command".
**Problem:** Since `disable-thp yes` (default), Valkey on Linux 3.5+ calls `prctl(PR_SET_THP_DISABLE)` for its own process when the kernel setting is `[always]`, and only warns if that fails. Readers are told a system-wide kernel change is mandatory when it usually is not.
**Evidence:** valkey@9.1.2:src/config.c:3309: `createBoolConfig("disable-thp", NULL, IMMUTABLE_CONFIG, server.disable_thp, 1, NULL, NULL),`
valkey@9.1.2:src/server.c:6855-6860:
```
    if (checkTHPEnabled(&err_msg) < 0) {
        server.thp_enabled = 1;
        if (THPDisable() == 0) {
            server.thp_enabled = 0;
        } else {
            serverLog(LL_WARNING, "WARNING %s", err_msg);
```
Live 9.1.2: `CONFIG GET disable-thp` returns `yes`.
**Fix:** Line 24: "Keep `disable-thp yes` (the default). Valkey then disables transparent huge pages for its own process. If the server logs a THP warning at startup, run `echo never > /sys/kernel/mm/transparent_hugepage/enabled` and restart Valkey." Add the same sentence before line 248. topics/admin.md:16 repeats the old advice and links here.

### F3 [minor] INACCURATE: Watchdog can be set in valkey.conf, and the minimum period depends on `hz` (`topics/latency.md:571`)
**Doc says:**
> Note that this feature cannot be enabled using the valkey.conf file, because it is designed to be enabled only in already running instances and only for debugging purposes.

> The minimum configurable period is 200 milliseconds.

**Problem:** `watchdog-period` is a normal (hidden) modifiable config, so the config file and command line accept it. The minimum is `2 * 1000 / hz`, which is 200 ms only at the default `hz 10`.
**Evidence:** valkey@9.1.2:src/config.c:3425: `createIntConfig("watchdog-period", NULL, MODIFIABLE_CONFIG | HIDDEN_CONFIG, 0, INT_MAX, ...`
valkey@9.1.2:src/debug.c:2525-2526:
```
        int min_period = (1000 / server.hz) * 2;
        if (server.watchdog_period < min_period) server.watchdog_period = min_period;
```
Live 9.1.2, started with `--watchdog-period 500`:
```
$ valkey-cli -p 20660 config get watchdog-period
watchdog-period
500
$ valkey-cli -p 20660 config set watchdog-period 50
OK
$ valkey-cli -p 20660 config get watchdog-period
watchdog-period
200
```
**Fix:** Delete line 571. Replace the last sentence of line 577 with: "Values below two server ticks (`2000 / hz` ms, 200 ms at the default `hz 10`) are raised to that minimum."

### F4 [minor] OUTDATED: EC2 advice names a retired instance generation (`topics/latency.md:23`)
**Doc says:**
> For EC2 users, make sure you use HVM based modern EC2 instances, like m3.medium. Otherwise fork() is too slow.

**Problem:** m3 is a previous-generation family. All current-generation EC2 instances are HVM/Nitro, and line 232 of the same page says slow fork "is hardly a problem". The item is no longer a checklist step.
**Evidence:** Page contradicts itself at topics/latency.md:229-232: "Modern hardware is pretty fast at copying the page table... As of 2024, this is hardly a problem."
**Fix:** Replace item 2 with: "Check fork time with `INFO stats` (`latest_fork_usec`) after a `BGSAVE`. On virtualized hosts without hardware-assisted page tables, fork can take hundreds of milliseconds on large instances." Or delete it.

### F5 [nit] EXAMPLE: `--intrinsic-latency` output omits the summary lines 9.1.2 prints (`topics/latency.md:84`)
**Doc says:**
> $ ./valkey-cli --intrinsic-latency 100
> Max latency so far: 1 microseconds.
> ...
> Max latency so far: 115 microseconds.

**Problem:** 9.1.2 ends the run with a total/average line and a worst-run ratio, which the prose could use. Both blocks (84-90, 102-109) also lack a language tag.
**Evidence:** Live 9.1.2:
```
Max latency so far: 46 microseconds.

41293772 total runs (avg latency: 0.0484 microseconds / 48.43 nanoseconds per run).
Worst run took 950x longer than the average latency.
```
**Fix:** Append the two summary lines to both examples and fence them as ```` ```console ````. The pattern of unfenced indented blocks applies to every example on the page (lines 46, 84, 102, 250, 287, 292, 305, 344, 415, 434, 507, 518, 525, 575, 583).
Tracked: valkey-io/valkey-doc#156

### F6 [nit] MISSING: Watchdog example relies on DEBUG, which is disabled by default (`topics/latency.md:601`)
**Doc says:**
> Note: in the example the **DEBUG SLEEP** command was used in order to block the server.

**Problem:** A reader reproducing this gets an error, because `enable-debug-command` defaults to `no`.
**Evidence:** Live 9.1.2:
```
ERR DEBUG command not allowed. If the enable-debug-command option is set to "local", you can run it from a local connection, otherwise you need to set this option in the configuration file, and then restart the server.
```
**Fix:** Append: "`DEBUG` requires `enable-debug-command` set to `yes` or `local`, which is off by default."

### F7 [nit] AI-NATIVE: Heading and intro do not say what the page covers (`topics/latency.md:14`)
**Doc says:**
> I've little time, give me the checklist

> This document will help you understand what the problem could be if you are experiencing latency problems with Valkey.

**Problem:** The heading carries no subject, and the intro does not list the causes covered or the audience.
**Fix:** Heading: "Latency troubleshooting checklist". Intro: "This page is for operators diagnosing slow Valkey responses. It covers measuring latency, the intrinsic latency of the host, and the common causes: network round trips, slow commands, fork, transparent huge pages, swapping, AOF fsync, mass key expiry, and the software watchdog." Cut line 36 ("And now for people with 15 minutes to spend, the details...") and lines 57-60.

### F8 [nit] STYLE: First-person and conversational voice (`topics/latency.md:18`)
**Doc says:**
> However I understand that we are busy people, so let's start with a quick checklist.

Also line 523 "so I use the following command", line 577 "In the above example I specified".
**Problem:** Single-author first person from the original Redis doc. Inconsistent with "we" elsewhere on the page.
**Fix:** Line 18-20: "Start with this checklist." Line 523: "To show only slow calls, filter the output:". Line 577: "This example logs a report when the server is blocked for 500 ms or longer."

### F9 [nit] STYLE: Typos and grammar pattern (`topics/latency.md:28`)
**Doc says:**
> use the following table for durability VS latency/performance tradeoffs

**Problem:** Line 28 calls a numbered list a "table". Other errors: 99 "In a virtualized environments", 140 "same host than the server", 241 "incurs to a big latency penalty", 245 "a few event loops runs", 284 "The first thing to do is to checking", 530 "Valkey evict expired keys", 538 "If the more than 25%", 540 "has no effects in the latency a Valkey instance".
**Fix:** 28 "use the following list for durability versus latency tradeoffs"; 99 "In virtualized environments"; 140 "same host as the server"; 241 "incurs a big latency penalty"; 245 "a few event loop iterations"; 284 "is to check"; 530 "Valkey evicts"; 540 "has no effect on the latency of a Valkey instance". 538 is replaced by F1.
