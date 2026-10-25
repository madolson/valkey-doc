VERDICT: NEEDS-WORK
COUNTS: blocker=1 minor=9 nit=5

# CLI (`cli`)

https://valkey.io/topics/cli/

The page is mostly correct in its prose but was refreshed from unstable and Redis-era material without rechecking against the release: it documents `VALKEYCLI_HOST`/`VALKEYCLI_PORT`, which 9.1.2 ignores, and many example outputs no longer match what 9.1.2 prints. Fix the env-var claim first; a user relying on it silently talks to 127.0.0.1:6379.

## Findings

### F1 [blocker] INACCURATE: `VALKEYCLI_HOST` and `VALKEYCLI_PORT` do not exist in 9.1.2 (`topics/cli.md:28`)
**Doc says:**
> Server hostname (default: `$VALKEYCLI_HOST` if set, otherwise 127.0.0.1).

> Server port (default: `$VALKEYCLI_PORT` if set, otherwise 6379).
**Problem:** Neither variable is read by 9.1.2. A user who exports `VALKEYCLI_HOST=prod-host` and runs `valkey-cli` connects to 127.0.0.1:6379 with no warning. The feature was added on unstable by valkey-io/valkey#3402 (commit baa5a48b5) and first ships in 9.2.0-rc1.
**Evidence:** `git -C ~/git/valkey grep -n 'VALKEYCLI_HOST\|VALKEYCLI_PORT' 9.1.2 -- src/` returns nothing. On unstable: `valkey@origin/unstable:src/valkey-cli.c:95: #define CLI_HOST_ENV "VALKEYCLI_HOST"`. `git merge-base --is-ancestor baa5a48b5 9.1.2` fails, and `git tag --contains baa5a48b5` lists 9.2.0-rc1. The 9.1.2 `--help`:
```
  -h <hostname>      Server hostname (default: 127.0.0.1).
  -p <port>          Server port (default: 6379).
```
**Fix:** `Server hostname (default: 127.0.0.1). Since Valkey 9.2, the default can be set with the VALKEYCLI_HOST environment variable.` Do the same for `-p` with `VALKEYCLI_PORT`.

### F2 [minor] INACCURATE: `HELP` category list has a wrong name and leaves out `@bitmap` (`topics/cli.md:704`)
**Doc says:**
> - `@sorted_set`
**Problem:** The group is named `sorted-set`. `HELP @sorted_set` prints nothing. `@bitmap` is a valid group and is not listed.
**Evidence:** Live 9.1.2: `valkey-cli help @sorted_set` prints an empty line. `valkey-cli help @sorted-set` lists BZMPOP and the other sorted-set commands. The groups in `COMMAND DOCS` are: bitmap, cluster, connection, generic, geo, hash, hyperloglog, list, pubsub, scripting, server, set, sorted-set, stream, string, transactions.
**Fix:** Replace `@sorted_set` with `@sorted-set` and add `@bitmap`.

### F3 [minor] OUTDATED: Cluster Manager table uses legacy "master" option names and leaves out atomic slot migration (`topics/cli.md:290`)
**Doc says:**
> --cluster-fix-with-unreachable-masters ... --cluster-use-empty-masters ... --cluster-master-id <arg> ... --cluster-only-masters
**Problem:** 9.1.2 prints `-primaries`/`-primary` names for these options and accepts the "master" names only as aliases. The page also leaves out `--cluster-use-atomic-slot-migration`. That option was added in 9.1.0 ("Add support for atomic slot migration to valkey-cli", #2755) and `reshard` and `rebalance` honor it.
**Evidence:** `valkey@9.1.2:src/valkey-cli.c:2734`: `!strcmp(argv[i], "--cluster-only-masters") || !strcmp(argv[i], "--cluster-only-primaries")`. Line 2740 handles `--cluster-master-id` / `--cluster-primary-id`, lines 2792-2793 handle `--cluster-use-empty-masters` / `--cluster-use-empty-primaries`, lines 2797-2798 handle `--cluster-fix-with-unreachable-masters` / `-primaries`, and lines 2800, 7900 and 8054 handle `--cluster-use-atomic-slot-migration`. The live `valkey-cli --cluster help` prints `--cluster-fix-with-unreachable-primaries`, `--cluster-use-empty-primaries` and `--cluster-only-primaries`. It also prints `--cluster-primaries-id`, which the parser does not accept (a server help-text bug).
**Fix:** Change the four options to `--cluster-fix-with-unreachable-primaries`, `--cluster-use-empty-primaries`, `--cluster-primary-id <arg>` and `--cluster-only-primaries`. Add `--cluster-use-atomic-slot-migration` under `reshard` and `rebalance`, with a note: "Available since Valkey 9.1. Moves slots with atomic slot migration instead of MIGRATE." Add the trailing line from the real help output: "For check, fix, reshard, del-node, set-timeout, info, rebalance, call, import, backup you can specify the host and port of any working node in the cluster."

### F4 [minor] INACCURATE: `-i` with `--bigkeys` throttles per 100 SCAN calls, not per SCAN (`topics/cli.md:786`)
**Doc says:**
> You can use -i 0.01 to sleep 0.01 sec per SCAN command (not usually needed).

> the `-i` option can be used in order to throttle the scanning process of the specified fraction of second for each `SCAN` command.
**Problem:** In `--bigkeys`, `--memkeys` and `--hotkeys`, the sleep runs once every 100 SCAN calls. That gives 100x less throttling than the page promises. It also contradicts the page's own option text at line 70 ("per 100 cycles").
**Evidence:** `valkey@9.1.2:src/valkey-cli.c:9429`: `if (config.interval && (scan_loops % 100) == 0) {`. Live 9.1.2 `--bigkeys` header:
```
# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).
```
**Fix:** At lines 814-820: "The `-i` option throttles the scan: the CLI sleeps for the given number of seconds after every 100 `SCAN` calls. For example, `-i 0.1` greatly reduces the load on a busy server." Replace the sample header with the real one above.

### F5 [minor] EXAMPLE: Many sample outputs no longer match what 9.1.2 prints (`topics/cli.md:1007`)
**Doc says (pattern):**
> `SYNC sent to master, writing 13256 bytes to '/tmp/dump.rdb'` (1007)
> `SYNC with master failed: -ERR Can't SYNC while not connected with my master` (1016)
> `SYNC with master, discarding 13256 bytes of bulk transfer...` / `SYNC done. Logging commands from master.` (1029-1030)
> `summary: Adds the specified elements to the specified HyperLogLog.` (722)
> `Reading messages... (press Ctrl-C to quit)` / `1) "PSUBSCRIBE"` (887-888)
> `[00.00%] Biggest string found so far 'key-419' with 3 bytes` and the summary block (790-808)
**Problem:** Readers who compare these with their terminal see different text. The old samples also keep the "master" wording, which the CLI itself no longer prints.
**Evidence:** Live 9.1.2 runs:
```
$ valkey-cli --rdb /tmp/docreview-cli/dump.rdb
sending REPLCONF capa eof
sending REPLCONF rdb-only 1
SYNC sent to primary, writing bytes of bulk transfer until EOF marker to '/tmp/docreview-cli/dump.rdb'
Transfer finished with success after 205628 bytes

$ valkey-cli --replica
sending REPLCONF capa eof
sending REPLCONF rdb-filter-only
Full resync with primary, discarding bytes of bulk transfer until EOF marker...
Full resync done after 212 bytes. Logging commands from primary.
sending REPLCONF ACK 0
"set","last_name","Enigk"

$ valkey-cli help pfadd
  PFADD key [element [element ...]]
  summary: Adds elements to a HyperLogLog key. Creates the key if it doesn't exist.
  since: 2.8.9
  group: hyperloglog

$ valkey-cli PSUBSCRIBE '*'     (tty)
1) "psubscribe"
2) "*"
3) (integer) 1

$ valkey-cli --bigkeys
[00.00%] Biggest string found so far '"item:3374"' with 6 bytes
-------- Summary --------
0 streams with 0 entries (00.00% of keys, avg size 0.00)
```
Error path: `valkey@9.1.2:src/valkey-cli.c:8745`: `"SYNC with primary failed: %s\n"`. The server reply is `valkey@9.1.2:src/replication.c:1119`: `"-NOMASTERLINK Can't SYNC while not connected with my master"`. The pub/sub banner is `valkey@9.1.2:src/valkey-cli.c:2303`: `"Reading messages... (press Ctrl-C to quit or any key to type command)"`.
**Fix:** Paste fresh 9.1.2 output into each block. For the `--rdb` failure use `SYNC with primary failed: -NOMASTERLINK Can't SYNC while not connected with my master`. At line 901, say that in interactive mode any key returns to the prompt and Ctrl-C quits.

### F6 [minor] BROKEN: Options list renders as run-on paragraphs with literal colons, and `<tag>` disappears (`topics/cli.md:27`)
**Doc says:**
> **`-X`**
> : Read <tag> argument from STDIN (see example below).
**Problem:** The site's Markdown renderer does not support the `term` / `: definition` syntax. Each of the roughly 70 options renders as one paragraph that runs into ": description". The raw `<tag>` goes out as an HTML tag, so browsers hide it. The text then reads "Read argument from STDIN". The page also has no `-X` example, although the option says "see example below".
**Evidence:** Live HTML:
```
<p><strong><code>-h</code></strong> <em>hostname</em>
: Server hostname (default: <code>$VALKEYCLI_HOST</code> if set, otherwise 127.0.0.1).</p>
...
: Read <tag> argument from STDIN (see example below).</p>
```
**Fix:** Turn the Options section into a two-column table (`Option | Description`) or a bullet list (`* `-h <hostname>`: ...`). Write `` `<tag>` `` in code. Add the `-X` example from `valkey-cli --help`: `valkey-cli -D "" --raw dump key > key.dump && valkey-cli -X dump_tag restore key2 0 dump_tag replace < key.dump`.

### F7 [minor] CONFUSING: Intrinsic latency conclusion says the opposite of what the measurement means (`topics/cli.md:994`)
**Doc says:**
> In the above case, the system cannot do better than 739 microseconds of worst case latency, so one can expect certain queries to occasionally run less than 1 millisecond.
**Problem:** A worst case of 739 µs means some queries can take up to about 0.74 ms because of the host alone. "Run less than 1 millisecond" reads as a speed guarantee.
**Fix:** "In this case, the host alone can add up to 739 microseconds to a request, so expect occasional queries to take close to 1 millisecond no matter how Valkey is tuned."

### F8 [minor] DUPLICATE: Intrinsic latency is explained in depth here and in latency.md (`topics/cli.md:962`)
**Doc says:**
> There is another pretty unusual latency tool implemented inside `valkey-cli`. It does not check the latency of a Valkey instance, but the latency of the computer running `valkey-cli`.
**Problem:** `topics/latency.md:75-111` covers the same concept, the same `--intrinsic-latency` command, the same "run it on the server host" warning and sample output. The two copies already drift: this page runs a 5 s test, latency.md runs 100 s.
**Fix:** Make latency.md canonical. In cli.md, cut lines 962-995 to a short paragraph: usage, "runs locally and does not connect to the server, so run it on the server host", and a link: "See [Diagnosing latency issues](latency.md) for how to interpret the result."

### F9 [minor] OUTDATED: History and rc-file sections document the legacy `REDISCLI_*` variables (`topics/cli.md:653`)
**Doc says:**
> It is possible to use a different history filename by setting the `REDISCLI_HISTFILE` environment variable

> by setting the `REDISCLI_RCFILE` environment variable (670)
**Problem:** The primary names are `VALKEYCLI_HISTFILE` and `VALKEYCLI_RCFILE`. The `REDISCLI_*` names are only fallbacks, used when the Valkey name is unset.
**Evidence:** `valkey@9.1.2:src/valkey-cli.c:86-90`:
```
#define CLI_HISTFILE_ENV "VALKEYCLI_HISTFILE"
#define OLD_CLI_HISTFILE_ENV "REDISCLI_HISTFILE"
#define CLI_RCFILE_ENV "VALKEYCLI_RCFILE"
#define OLD_CLI_RCFILE_ENV "REDISCLI_RCFILE"
```
`getDotfilePath()` (line 359) reads the new name first and the old name only as a fallback.
**Fix:** Use `VALKEYCLI_HISTFILE` and `VALKEYCLI_RCFILE`, and add: "The legacy `REDISCLI_HISTFILE` and `REDISCLI_RCFILE` names are still honored when the Valkey names are unset."

### F10 [minor] MISSING: No word on how to get `valkey-cli` (`topics/cli.md:7`)
**Doc says:**
> **`valkey-cli`** [_OPTIONS_] [_cmd_ [_arg_...]]
**Problem:** The page starts with usage and never says where the binary comes from: it is built with the server (`src/valkey-cli`) and shipped in distro packages and container images. It also doesn't say that TLS and RDMA options exist only in builds with that support. The 9.1.2 help on a non-TLS build omits `--tls`, `--cacert` and the rest, and `--rdma` is compiled in only with `USE_RDMA` (`valkey@9.1.2:src/valkey-cli.c:2806,2827`).
**Fix:** Add a short "Getting valkey-cli" paragraph: it ships with every Valkey server build and package, link to [Installation](installation.md), and note "TLS options require a build with `BUILD_TLS=yes`; `--rdma` requires `BUILD_RDMA`."
Tracked: valkey-io/valkey-doc#470, valkey-io/valkey-doc#200

### F11 [nit] OUTDATED: Nonexistent version "Valkey 3.2" (`topics/cli.md:537`)
**Doc says:**
> the debugging facility of Lua scripting, available with Valkey 3.2 onwards.
**Problem:** Valkey's first release was 7.2.4. The debugger predates Valkey.
**Fix:** Delete "available with Valkey 3.2 onwards". Every Valkey release has the debugger.

### F12 [nit] STYLE: Typos and stray characters (`topics/cli.md:103`)
**Doc says (pattern):** `Prefer IPv6 over IPv4 on DNS lookup.'` (103, stray quote); `just process CTRL-C` (901); `Valkey' LRU` (1063); `therefor` (1097); `all other states about the connection is lost` (627).
**Fix:** `lookup.`; `press Ctrl-C`; `Valkey's LRU`; `therefore`; `all other connection state is lost`.

### F13 [nit] STYLE: Setext H1 headings appear after H2 sections (`topics/cli.md:560`)
**Doc says:**
> Interactive mode
> ===

> Special modes of operation (731)
**Problem:** These render as `<h1>` in the middle of the page, under H2s like "Options", so the outline is inverted. The live HTML shows `<h1 id="interactive-mode">` and `<h1 id="special-modes-of-operation">`.
**Fix:** Make them `## Interactive mode` and `## Special modes of operation`, and demote their subsections to `###`. Rename "Description" (line 11) to "What valkey-cli does".

### F14 [nit] STYLE: Fenced blocks without a language tag, and a mis-indented fence (`topics/cli.md:280`)
**Doc says:** fences at 280, 379 and 389 open with a bare ```` ``` ````. Line 391 closes with ```` ``` ```` indented by one space.
**Fix:** Tag them ```` ```text ```` (280) and ```` ```console ```` (379, 389), and remove the leading space at 391.

### F15 [nit] AI-NATIVE: Front matter description and intro don't say what the page covers or who it is for (`topics/cli.md:4`)
**Doc says:**
> description: >
>     Valkey command line interface
**Problem:** The description just restates the title. The intro (13-23) lists only three of the special modes and never names the audience.
**Fix:** `description: "Reference and guide for valkey-cli: options, interactive mode, scripting, cluster management, and diagnostic modes such as --bigkeys, --latency, --rdb, and --replica."` First sentence: "`valkey-cli` is the command-line client that ships with Valkey. Operators and developers use it to run commands, script against a server, manage clusters, and diagnose latency, memory, and replication."

## Unverified
- topics/cli.md:592-641: interactive `CONNECT`, auto-reconnect with DB re-select, `5 INCR` repeat prefix, `CLEAR`, and `:set hints`. These need an interactive linenoise session. I confirmed in source that the handlers and the `not connected> ` prompt exist (`src/valkey-cli.c:3337`, `:set hints` at line 959), but I did not drive them live.
- topics/cli.md:105-137: TLS options (`--sni`, `--cacertdir`, `--insecure`, `--tls-ciphersuites`). The parser has them under `#ifdef USE_OPENSSL` (`src/valkey-cli.c:2806-2826`), but I could not run them because the test build has no TLS.
- topics/cli.md:1080-1106: `--lru-test` sample figures. I did not run a 10M-key simulation. The output format string matches (`src/valkey-cli.c:9853`).
