VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=6 nit=6

# Benchmarking tool (`benchmark`)

https://valkey.io/topics/benchmark/

The page is a hand-copied, older version of `valkey-benchmark --help`. It is missing the options added in 8.1, 9.0 and 9.1, and its sample output predates the current output format. The most important fix is to resync the Options section with 9.1.2 (`--duration`, `--warmup`, `--sequential`, `__rand_1st__`, and the rest). PR #441 already does part of this.

## Findings

### F1 [minor] MISSING: Options and placeholders added in 8.1 to 9.1 are not documented (`topics/benchmark.md:20`)
**Doc says:**
> In command arguments, the following placeholders are substituted: `__rand_int__` ... `__data__` ... `{tag}`

The Options list (lines 32 to 190) has no `--duration`, `--warmup`, `--sequential`, `--num-functions`, `--num-keys-in-fcall`, `--fuzz`, `--fuzz-mode` or `--fuzz-loglevel`.
**Problem:** 9.1.2 has eight options and nine placeholders that the page does not mention. The `__rand_int__` (line 23) and `-P` (line 114) descriptions also leave out behavior that changes results: each `__rand_int__` occurrence gets a different value, and with a multi-command sequence `-P` counts whole sequences.
**Evidence:** `valkey-benchmark --help` from the 9.1.2 binary:
```
__rand_1st__        Like __rand_int__ but multiple occurrences will have the same
                    value. __rand_2nd__ through __rand_9th__ are also available.
 --duration <seconds>
                    Run benchmark for specified number of seconds
                    (mutually exclusive with -n)
 --warmup <seconds> Run benchmark for specified warmup period before
                    recording data
 --sequential       Modifies the -r argument to replace the string __rand_int__
                    with 12 digit numbers sequentially instead of randomly.
 -P <numreq>        ... When multiple commands are specified on the command line,
                    then the full command sequence counts as one and -P controls
                    the number of times the command sequence is sent in each
                    pipeline.
 --num-functions <num> ... (default 10).
 --num-keys-in-fcall <num> ... (default 1)
 --fuzz             Enable fuzzy mode to generate random commands. WARNING: Recommended for testing only, not for use with production data.
 --fuzz-mode <modes> ...
 --fuzz-loglevel <level> ...
```
First tag containing each option in valkey@`src/valkey-benchmark.c`: `--num-functions` and `--num-keys-in-fcall` 8.1.0; `--sequential` and `__rand_1st__` 9.0.0; `--duration`, `--warmup` and `--fuzz*` 9.1.0. Live check: `valkey-benchmark -n 10 --duration 1` prints `Options -n and --duration are mutually exclusive.`
**Fix:** Add each option with its help text and a "since" note: `--num-functions` and `--num-keys-in-fcall` (8.1), `--sequential` (9.0), `--duration`, `--warmup` and `--fuzz`/`--fuzz-mode`/`--fuzz-loglevel` (9.1). Add `__rand_1st__` to `__rand_9th__` (9.0) to the placeholder list. Append to `__rand_int__`: "Multiple occurrences within the command get different values." Append the multi-command sentence above to `-P`. Put the `--fuzz` warning against production data in the text.
Tracked: valkey-io/valkey-doc#414, valkey-io/valkey-doc#441

### F2 [minor] EXAMPLE: Transaction example has a C string tail pasted in and does not run as written (`topics/benchmark.md:227`)
**Doc says:**
> `incr counter ';' exec\n\n");`
**Problem:** `\n\n");` was copied from the C usage string. When pasted, the unbalanced `"` makes the shell wait for more input. The live page renders it the same way (`exec\n\n&quot;);`).
**Evidence:** valkey@9.1.2:src/valkey-benchmark.c usage ends with `incr counter ';' exec\n"`. The command without the artifact runs on 9.1.2:
```
multi ; set key:__rand_int__ __data__ ; incr counter ; exec: 249999.98 requests per second, p50=0.383 msec
```
**Fix:** Change line 227 to `                          incr counter ';' exec`.

### F3 [minor] EXAMPLE: Sample output does not match what 9.1.2 prints (`topics/benchmark.md:236`)
**Doc says:**
> `SET: 74239.05 requests per second`

and lines 261 to 271:
> `99.76% `<=` 1 milliseconds` ... `72144.87 requests per second`
**Problem:** In quiet mode 9.1.2 appends `p50=`. The full report now includes host config lines, a percentile distribution, a cumulative distribution and a summary table. The old `<=` lines are gone. Lines 267 to 270 also show literal backticks on the live page (`` `&lt;=` ``) and repeat the 100.00% line twice.
**Evidence:** 9.1.2 run:
```
SET: 68493.15 requests per second, p50=0.375 msec
====== SET ======
  200000 requests completed in 3.05 seconds
  50 parallel clients
  3 bytes payload
  keep alive: 1
  host configuration "save": 
  host configuration "appendonly": no
  multi-thread: no

Latency by percentile distribution:
0.000% <= 0.127 milliseconds (cumulative count 1)
50.000% <= 0.367 milliseconds (cumulative count 121208)
...
Cumulative distribution of latencies:
```
**Fix:** Regenerate the output on lines 236 to 237, 244, 261 to 271 and 292 to 293 from a 9.1.2 run. For the long `-r` example, cut the run with `...` after the percentile section rather than showing the old format.

### F4 [minor] BROKEN: Definition-list syntax renders as literal colons on the live page (`topics/benchmark.md:22`)
**Doc says:**
> ``**`-h`** _hostname_``
> `: Server hostname (default 127.0.0.1)`
**Problem:** The site's renderer does not support `term` / `: definition` syntax, so every option and placeholder (lines 22 to 190) shows up as a paragraph starting with ": ". The `--rfr` mode list and its follow-up sentence (lines 86 to 93) fall out of the option entry and look like top-level text.
**Evidence:** Live HTML:
```
<p><strong><code>-h</code></strong> <em>hostname</em>
: Server hostname (default 127.0.0.1)</p>
...
<p><strong>no</strong> - sends read requests to primaries only (default).</p>
```
`topics/cli.md` and `topics/server.md` use the same syntax, so this affects those pages too.
**Fix:** Turn the Options and placeholder sections into a table (`| Option | Description | Default |`), or a bulleted list such as ``- **`-h`** _hostname_: Server hostname (default 127.0.0.1)``. Make the `--rfr` modes a nested list under their option.

### F5 [minor] MISSING: The list of default test names for `-t` is not given (`topics/benchmark.md:129`)
**Doc says:**
> Only run the comma separated list of tests. The test names are the same as the ones produced as output.
**Problem:** The only way to find valid `-t` names is to run the full suite first. Some names are not command names (`lrange_100`, `function_load`, `fcall`).
**Evidence:** valkey@9.1.2:src/valkey-benchmark.c `test_is_selected(...)` names (lines 2505 to 2530 show `lrange` selects all four lrange tests): `ping set get incr lpush rpush lpop rpop sadd hset spop zadd zpopmin lrange_100 lrange_300 lrange_500 lrange_600 mset mget xadd function_load fcall`. Note that `ping` selects both PING_INLINE and PING_MBULK.
**Fix:** Add to the `-t` entry: "Available tests: `ping`, `set`, `get`, `incr`, `lpush`, `rpush`, `lpop`, `rpop`, `sadd`, `hset`, `spop`, `zadd`, `zpopmin`, `lrange` (shorthand for all of the following), `lrange_100`, `lrange_300`, `lrange_500`, `lrange_600`, `mset`, `mget`, `xadd`, `function_load`, `fcall`."

### F6 [minor] OUTDATED: 2010-era Redis measurements are presented as Valkey results (`topics/benchmark.md:403`)
**Doc says:**
> Here are some results of 4 KB SET benchmark for 3 server CPUs (AMD Istanbul, Intel Nehalem EX, and Intel Westmere)

and line 412:
> Valkey has already been benchmarked at more than 60000 connections, and was still able to sustain 50000 q/s in these conditions. As a rule of thumb, an instance with 30000 connections can only process half the throughput achievable with 100 connections.
**Problem:** These figures and the three charts (`Data_size.png`, `NUMA_chart.gif`, `Connections_chart.png`) come from Redis measurements on CPUs from about 2010. Valkey did not exist then. The "half the throughput" rule of thumb predates I/O threads, so readers will size systems on numbers that were never measured on Valkey. Line 424 also links to a 2011 redis-db Google Groups thread.
**Evidence:** The CPU models named on line 404. The `redis-db` group URL on line 424. There is no Valkey source or benchmark for the 60000-connection claim.
**Fix:** Drop the specific numbers. Keep the guidance: "Throughput drops as the connection count grows. Measure with the `-c` value you expect in production." Label the charts as historical Redis measurements, or remove them. Replace the redis-db link with the Linux kernel RPS/RFS docs: https://docs.kernel.org/networking/scaling.html.

### F7 [nit] AI-NATIVE: No opening paragraph, weak description, and a "Notes" heading that says nothing (`topics/benchmark.md:3`)
**Doc says:**
> `Using the valkey-benchmark utility on a Valkey server`

The page starts at `## Usage`, and line 297 is `## Notes`.
**Problem:** Nothing says what the page covers or who it is for. "Notes" holds the most valuable Valkey-specific guidance (pitfalls, performance factors) under a heading that describes none of it.
**Fix:** Description: "Measure Valkey throughput and latency with valkey-benchmark: options, examples, and how to get results you can reproduce." Add an opening paragraph before Usage: "`valkey-benchmark` ships with Valkey and simulates N clients sending commands at the same time. This page lists its options and shows typical runs, then explains how to set up a benchmark whose results you can compare." Rename `## Notes` to `## Getting meaningful results`.

### F8 [nit] OUTDATED: The EVAL example uses `redis.call` (`topics/benchmark.md:218`)
**Doc says:**
> `valkey-benchmark -r 10000 -n 10000 eval 'return redis.call("ping")' 0`
**Problem:** It works because of the compatibility alias, but the tool's own help and line 243 of this page use `server.call`.
**Evidence:** 9.1.2 `--help`: `$ valkey-benchmark -r 10000 -n 10000 eval 'return server.call("ping")' 0`
**Fix:** Change `redis.call` to `server.call`.

### F9 [nit] MISSING: Does not say that TLS and RDMA options depend on build flags (`topics/benchmark.md:146`)
**Doc says:**
> **`--tls`** : Establish a secure TLS connection. ... **`--rdma`** : Establish an RDMA connection.
**Problem:** The binary only accepts these options when built with TLS or RDMA. Without them they are missing from `--help` and rejected as unknown, which confuses readers.
**Evidence:** valkey@9.1.2:src/valkey-benchmark.c:1777 `#ifdef USE_OPENSSL` and :1806 `#ifdef USE_RDMA` wrap the parsing. `--help` from a build without TLS lists no TLS options.
**Fix:** Before `--tls`, add: "TLS options are available only when Valkey is built with `BUILD_TLS=yes`. `--rdma` requires `BUILD_RDMA`. See [RDMA](RDMA.md)."

### F10 [nit] STYLE: First-person voice and filler (`topics/benchmark.md:253`)
**Doc says:**
> For instance if I want to run one million SET operations, using a random key for every operation out of 100k possible keys, I'll use the following command line:

and line 301:
> The first point is obvious: the golden rule of a useful benchmark is to only compare apples and apples.
**Problem:** The first-person voice and "obvious" add no information and do not match the rest of the docs.
**Fix:** "To run one million SET operations against random keys drawn from 100,000 possible keys:" and "Only compare like with like."

### F11 [nit] EXAMPLE: Code blocks are indented with no language tag (`topics/benchmark.md:197`)
**Doc says:**
> `    $ valkey-benchmark`
**Problem:** Every example (lines 197 to 293) is an indented code block, which renders as `data-lang="plain"`. The images on lines 394, 408 and 418 are also indented inside list items.
**Fix:** Use fenced blocks tagged `` ```sh `` for commands and `` ```text `` for output.

### F12 [nit] BROKEN: Third-party tool links point to stale owners and social accounts (`topics/benchmark.md:472`)
**Doc says:**
> [memtier_benchmark](https://github.com/redislabs/memtier_benchmark) from [Redis Ltd.](https://twitter.com/RedisInc)
**Problem:** `github.com/redislabs/memtier_benchmark` redirects to `github.com/redis/memtier_benchmark`. The Twitter links for vendors add nothing and point to a renamed service. "Yahoo @Yahoo" on line 474 is garbled.
**Evidence:** `curl -sIL` gives `200 https://github.com/redis/memtier_benchmark`.
**Fix:** Change the link to `https://github.com/redis/memtier_benchmark` and drop the vendor Twitter links: "* [memtier_benchmark](https://github.com/redis/memtier_benchmark) from Redis Ltd. ...", "* [rpc-perf](https://github.com/twitter/rpc-perf) ...", "* [YCSB](https://github.com/brianfrankcooper/YCSB) from Yahoo ...".

## Unverified
- topics/benchmark.md:383: "unix domain sockets can achieve around 50% more throughput than the TCP/IP loopback". No Valkey measurement exists, and I did not benchmark it.
- topics/benchmark.md:311: the threading description. It looks broadly right for 9.1.2 I/O threads. Issue valkey-io/valkey-doc#196 tracks threading wording across the docs.
- Unstable changes src/valkey-benchmark.c by 442 lines over 9.1.2. I did not review them for options that will need docs at the next release.
