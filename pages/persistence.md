VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=8 nit=5

# Persistence (`persistence`)

https://valkey.io/topics/persistence/

Mostly accurate, but the AOF recovery sections are Redis-era. They show stale single-file log output, imply that some Valkey versions can't load a truncated AOF, and point `valkey-check-aof` at a single file instead of the manifest. Biggest fix: rewrite the truncated/corrupted AOF sections around multi-part AOF and state the real defaults (save points, rewrite triggers, RDB preamble).

## Findings

### F1 [minor] INACCURATE: everysec can lose up to two seconds, not one (`topics/persistence.md:30`)
**Doc says:**
> fsync is performed using a background thread and the main thread will try hard to perform writes when no fsync is in progress, so you can only lose one second worth of writes.

Line 149 repeats it: "you may lose 1 second of data if there is a disaster."
**Problem:** When a background fsync is slow, the main thread holds the write back for up to 2 seconds before writing anyway. The worst case is about 2 seconds of writes, not 1.
**Evidence:** valkey@9.1.2:src/aof.c:1225-1232
```c
} else if (server.mstime - server.aof_flush_postponed_start < 2000) {
    /* We were already waiting for fsync to finish, but for less
     * than two seconds this is still ok. Postpone again. */
    return;
}
/* Otherwise fall through, and go write since we can't wait
 * over two seconds. */
```
**Fix:** Line 30: "...so you typically lose at most about one second of writes, and up to two seconds if the disk is slow to fsync." Line 149: "you may lose 1 to 2 seconds of data if there is a disaster."

### F2 [minor] OUTDATED: "Older versions of Valkey may not recover" from a truncated AOF (`topics/persistence.md:164`)
**Doc says:**
> The latest major versions of Valkey will be able to load the AOF anyway ... Older versions of Valkey may not recover, and may require the following steps:
> `$ valkey-check-aof --fix <filename>`
**Problem:** Every Valkey release ships with `aof-load-truncated yes`, so no Valkey version fails on a truncated AOF by default. This version split comes from Redis 3.0. The manual steps only apply when `aof-load-truncated` is `no`. The tool should also be pointed at the manifest, which is what the server's own error message recommends.
**Evidence:** valkey@9.1.2:src/config.c:3291 `createBoolConfig("aof-load-truncated", ..., server.aof_load_truncated, 1, ...)`. valkey@9.1.2:src/aof.c:1755: `"1) Make a backup of your AOF file, then use ./valkey-check-aof --fix <filename.manifest>. "`. Tool usage on 9.1.2:
```
Usage: valkey-check-aof [--fix|--truncate-to-timestamp $timestamp] <file.manifest|file.aof>
```
**Fix:** Replace lines 164 and 181 with: "Valkey loads the AOF anyway, discarding the last incomplete command, because `aof-load-truncated` defaults to `yes`. If you set `aof-load-truncated no`, the server refuses to start, and you need to:" Change the command to `valkey-check-aof --fix appendonlydir/appendonly.aof.manifest`.

### F3 [minor] EXAMPLE: AOF log samples don't match 9.1.2 output (`topics/persistence.md:168`)
**Doc says:**
```
* Reading RDB preamble from AOF file...
* Reading the remaining AOF tail...
# !!! Warning: short read while loading the AOF file !!!
# !!! Truncating the AOF at offset 439 !!!
# AOF loaded anyway because aof-load-truncated is enabled
```
and at line 199: `# Bad file format reading the append only file: make a backup of your AOF file, then use ./valkey-check-aof --fix <filename>`
**Problem:** Those lines come from the old single-file AOF format. On 9.1.2 with multi-part AOF, the log names the base and incr files and recommends `<filename.manifest>`.
**Evidence:** Real 9.1.2 output after truncating `appendonly.aof.3.incr.aof`:
```
* Reading RDB base file on AOF loading...
* RDB is base AOF
* DB loaded from base file appendonly.aof.3.base.rdb: 0.000 seconds
# !!! Warning: short read while loading the AOF file appendonly.aof.3.incr.aof!!!
# !!! Truncating the AOF appendonly.aof.3.incr.aof at offset 107 !!!
# AOF appendonly.aof.3.incr.aof loaded anyway because aof-load-truncated is enabled
```
After corrupting a byte in the incr file (server exits):
```
# Bad file format reading the append only file appendonly.aof.3.incr.aof: make a backup of your AOF file, then use ./valkey-check-aof --fix <filename.manifest>
```
**Fix:** Replace both blocks with the output above. Use a `text` language tag.

### F4 [minor] CONFUSING: AOF rewrite explained four times (`topics/persistence.md:126`)
**Doc says:**
> The rewrite is completely safe. While Valkey continues appending to the old file, a completely new one is produced ... Valkey switches the two and starts appending to the new one.
**Problem:** Line 32 and lines 126-129 are near-identical. Both describe the old single-file "switches the two" model, which then contradicts the multi-part description at 135-140 and again at 213-231. A reader can't tell which model is current. Line 131 ("Valkey supports an interesting feature") and line 133 ("automatically trigger log rewriting automatically") add nothing.
**Fix:** Delete lines 126-129 and 131. Reword line 32 to: "Valkey can rewrite the AOF in the background when it grows too large. The child writes a new base file while the parent writes new commands to a new incremental file, so a failed rewrite loses nothing." Keep 135-140 and 213-231 as the single explanation and merge them into one section under "Log rewriting".

### F5 [minor] MISSING: defaults and limits not stated (`topics/persistence.md:58`)
**Doc says:**
> By default Valkey saves snapshots of the dataset on disk ... For example, this configuration ... `save 60 1000`

Line 133: "(see the example configuration file for more information)". Line 140: "failed AOF rewrites are retried at a slower and slower rate."
**Problem:** The page never gives the defaults that decide real behavior: the default save points, the rewrite triggers, the base-file format, the AOF directory name, or the rewrite backoff. All of these are facts readers want to quote.
**Evidence:** Live 9.1.2: `CONFIG GET save` returns `3600 1 300 100 60 10000`. valkey@9.1.2:valkey.conf:1683-1684 `auto-aof-rewrite-percentage 100`, `auto-aof-rewrite-min-size 64mb`. src/config.c:3292 `aof-use-rdb-preamble` default 1. src/config.c:3339 `appenddirname` default `"appendonlydir"`. src/aof.c:857-858 `AOF_REWRITE_LIMIT_THRESHOLD 3`, `AOF_REWRITE_LIMIT_MAX_MINUTES 60`, delay starting at 1 minute and doubling (aof.c:879).
**Fix:** Line 58: "By default Valkey saves a snapshot after 3600 s if at least 1 key changed, after 300 s if 100 changed, or after 60 s if 10000 changed (`save 3600 1 300 100 60 10000`)." Line 133: "Valkey starts a rewrite automatically when the AOF has grown by `auto-aof-rewrite-percentage` (default 100) since the last rewrite and is at least `auto-aof-rewrite-min-size` (default 64mb)." Line 115: "The base file is in RDB format when `aof-use-rdb-preamble` is `yes` (the default), otherwise AOF format. Files live in `appenddirname` (default `appendonlydir`) under `dir`." Line 140: "After 3 consecutive failures, retries are delayed by 1 minute, doubling up to 60 minutes."

### F6 [minor] OUTDATED: discouraging AOF alone (`topics/persistence.md:50`)
**Doc says:**
> There are many users using AOF alone, but we discourage it since to have an RDB snapshot from time to time is a great idea for doing database backups, for faster restarts, and in the event of bugs in the AOF engine.
**Problem:** With the default `aof-use-rdb-preamble yes`, every AOF base file is already an RDB snapshot (`appendonly.aof.N.base.rdb` on 9.1.2). The "faster restarts" argument therefore mostly falls away, and the page itself documents AOF backups at line 290. Line 20 ("RDB allows faster restarts ... compared to AOF") has the same problem.
**Evidence:** The 9.1.2 run produced `appendonly.aof.3.base.rdb` and logged `RDB is base AOF`. src/config.c:3292.
**Fix:** Replace with: "AOF alone is a valid choice. Its base file is an RDB snapshot by default, so restarts load mostly RDB data. Keep RDB as well if you want a single-file artifact for backups and transfer." Line 20: "RDB restarts faster than AOF when the AOF has a large incremental tail."
Tracked: valkey-io/valkey-doc#360

### F7 [minor] AI-NATIVE: duplicate, misplaced and vague headings (`topics/persistence.md:80`)
**Doc says:**
> `### How it works` (line 80, under "No persistence") and `### How it works` (line 211, under the AOF section). Also `Ok, so what should I use?` (line 41) and `### How I can switch to AOF, if I'm currently using dump.rdb snapshots?` (line 233).
**Problem:** The two identical headings render as `#how-it-works` and `#how-it-works-1`, so neither tells the reader or a search result what it covers. The first one describes RDB snapshotting but sits under "No persistence".
**Evidence:** Live page anchors: `<h3 id="how-it-works"`, `<h3 id="how-it-works-1"`.
**Fix:** Move lines 80-92 to sit under "Snapshotting" and retitle it "How RDB snapshotting works". Retitle line 211 "How AOF rewriting works". Line 41: "## Choosing RDB, AOF, or both" in ATX style. Line 233: "### Switching from RDB to AOF".

### F8 [minor] STYLE: security advice recommends a passphrase-less SSH key with full access (`topics/persistence.md:328`)
**Doc says:**
> generate a ssh client key without passphrase, then add it in the `authorized_keys` file of your small VPS.
**Problem:** As written, this gives the Valkey host an unrestricted shell login on the backup host. If the Valkey host is compromised, every backup can be deleted.
**Fix:** "generate a dedicated SSH key without a passphrase and restrict it in `authorized_keys` to file upload only (for example `command="rrsync -wo /backups",restrict`), so a compromised Valkey host cannot delete existing backups."

### F9 [nit] MISSING: BGSAVE during an AOF rewrite is rejected unless SCHEDULE is used (`topics/persistence.md:260`)
**Doc says:**
> Valkey makes sure to avoid triggering an AOF rewrite when an RDB snapshotting operation is already in progress, or allowing a `BGSAVE` while the AOF rewrite is in progress.
**Problem:** The page explains BGREWRITEAOF scheduling but not the BGSAVE side, which returns an error.
**Evidence:** 9.1.2:
```
ERR Another child process is active (AOF?): can't BGSAVE right now. Use BGSAVE SCHEDULE in order to schedule a BGSAVE whenever possible.
```
**Fix:** Add: "`BGSAVE` during a rewrite returns an error. Use `BGSAVE SCHEDULE` to run it once the rewrite finishes."

### F10 [nit] CONFUSING: AOF advertised as having "no corruption problems" (`topics/persistence.md:31`)
**Doc says:**
> The AOF log is an append-only log, so there are no seeks, nor corruption problems if there is a power outage.
**Problem:** This contradicts the page's own section "What should I do if my AOF gets corrupted?" (line 191).
**Fix:** "Because the AOF is append-only, a power outage usually leaves at most a truncated last command, which Valkey discards on load."

### F11 [nit] STYLE: code blocks without language tags (`topics/persistence.md:66`)
**Doc says:** Indented blocks at lines 66, 76, 107, 186 and untagged fences at 168, 197.
**Problem:** These render without highlighting and with no stated language.
**Fix:** Use fenced blocks: `conf` for 66, 76, 107, `sh` for 186, `text` for 168 and 197.

### F12 [nit] STYLE: wording errors (`topics/persistence.md:19`)
**Doc says:** "maximizes Valkey performances" (19), "The incremental files contains" (116), "a new increments AOF file" (223), "Profit!" (231), "make sure to read the following sentence: **Make Sure to Backup Your Database**" (276), "check what is the difference between two files" (188).
**Fix:** "maximizes Valkey performance", "incremental files contain", "a new incremental AOF file", drop "Profit!", "Back up your database." (drop the lead-in), "check the difference between the two files".

### F13 [nit] AI-NATIVE: weak intro and generic disaster-recovery filler (`topics/persistence.md:13`)
**Doc says:**
> To learn more about how to evaluate your Valkey persistence strategy, read on.

Lines 324-335: "We'll review the most interesting disaster recovery techniques that don't have too high costs ... get a small VPS in a place that is very far from you".
**Problem:** The intro doesn't say what the page covers or who it is for. The disaster-recovery section is generic backup advice with nothing Valkey-specific beyond "copy the RDB".
**Fix:** Replace line 13 with: "This page is for operators choosing and running Valkey persistence. It covers how RDB and AOF work, their defaults and tradeoffs, recovering a damaged AOF, switching from RDB to AOF, and backing up each format." Cut lines 324-329 down to: "Copy RDB snapshots (or a hard-linked AOF directory, see above) off-site, encrypted, to at least two providers. Verify the size and checksum after each transfer, and alert when a transfer fails."
