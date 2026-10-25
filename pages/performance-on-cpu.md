VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=6

# CPU profiling (`performance-on-cpu`)

https://valkey.io/topics/performance-on-cpu/

The perf and bcc commands are fine. The Valkey-specific parts are not: the build step tells readers to add a flag the build already sets, and the funccount example names functions that were removed with the old I/O threads code. Fix the build prerequisites section first.

## Findings

### F1 [minor] INACCURATE: Build step is redundant and doesn't match the prose around it (`topics/performance-on-cpu.md:50`)
**Doc says:**
> It's important that you ensure that:
> - we still run with optimizations to get an accurate representation of production run times, meaning we will keep: `-O3`
>
> You can do it as follows within valkey main repo:
>
>     $ make SERVER_CFLAGS="-g"

**Problem:** The default build already includes `-g -ggdb`, `-O3`, `-flto` and `-fno-omit-frame-pointer`, so `SERVER_CFLAGS="-g"` changes nothing. The one-item list talks about `-O3`, and the command below it adds `-g`, so readers can't tell what "it" means. The page also never says that `make install` (plain `install`, no `-s`) keeps symbols, or that LTO is on by default and inlines functions across files, which changes what shows up in stacks.
**Evidence:** valkey@9.1.2:src/Makefile:23-33
```
OPTIMIZATION?=-O3
ifeq ($(OPTIMIZATION),-O3)
	ifeq (clang,$(CLANG))
		OPTIMIZATION+=-flto
	else
		OPTIMIZATION+=-flto=auto -ffat-lto-objects
...
ifneq ($(OPTIMIZATION),-O0)
	OPTIMIZATION+=-fno-omit-frame-pointer
```
valkey@9.1.2:src/Makefile:139,142
```
FINAL_CFLAGS=$(STD) $(WARN) $(OPT) $(DEBUG) $(CFLAGS) $(SERVER_CFLAGS)
DEBUG=-g -ggdb
```
**Fix:** Replace lines 39-55 with: "A default `make` builds Valkey with `-O3 -flto -fno-omit-frame-pointer -g -ggdb`. That gives an optimized binary with frame pointers and debug symbols, which is what profilers need, so no extra flags are required. Don't build with `OPTIMIZATION=-O0` or `make noopt`: that drops `-fno-omit-frame-pointer` and doesn't reflect production. Don't strip the binary. If you build with custom `CFLAGS`, keep `-fno-omit-frame-pointer` and `-g`. Modules must be built with frame pointers too."

### F2 [minor] EXAMPLE: funccount output lists functions that don't exist in Valkey 9.1.2 (`topics/performance-on-cpu.md:179`)
**Doc says:**
> clientInstallWriteHandler                 388
> ...
> handleClientsWithPendingReadsUsingThreads      735
> handleClientsWithPendingWritesUsingThreads      735

**Problem:** This sample output is from Redis-era threading code. Three of the seven functions shown aren't in 9.1.2, so a reader running the command gets a different list and may think their symbols are broken.
**Evidence:** `git -C ~/git/valkey grep -n <name> 9.1.2 -- 'src/*.c'` returns nothing for `handleClientsWithPendingReadsUsingThreads`, `handleClientsWithPendingWritesUsingThreads`, or `clientInstallWriteHandler`. The I/O thread entry points in 9.1.2 are different (valkey@9.1.2:src/io_threads.h:27-28: `int trySendReadToIOThreads(client *c);`, `int trySendWriteToIOThreads(client *c);`). `call`, `handleClientsWithPendingWrites`, `postponeClientRead` and `prepareClientToWrite` still exist (src/server.c:3830, src/networking.c:3348, 4388, 474).
**Fix:** Re-capture the output against a 9.1.2 server. Or delete the three missing rows, fix the "Tracing 64 functions" count, and add a note that the function list depends on the version and on `io-threads`.

### F3 [minor] INACCURATE: Says perf script generates perf.data (`topics/performance-on-cpu.md:125`)
**Doc says:**
> By default, perf script will generate a perf.data file in the current working directory.

**Problem:** `perf script` reads `perf.data`, it doesn't write it. This line was copied from the `perf record` paragraph at line 99.
**Evidence:** perf-script(1): "-i, --input=: Input file name. (default: perf.data unless stdin is a fifo)". The command at line 121 writes to stdout (`perf script > valkey.perf.stacks`).
**Fix:** "By default, perf script reads perf.data from the current working directory. See the [perf script](https://man7.org/linux/man-pages/man1/perf-script.1.html) documentation for advanced usage."

### F4 [minor] BROKEN: Empty duplicate heading (`topics/performance-on-cpu.md:167`)
**Doc says:**
> ### Visualizing the recorded profile information using Flame Graphs

**Problem:** This heading has no content under it. The H2 at line 169 follows directly. The live page renders it as an empty section with anchor `visualizing-the-recorded-profile-information-using-flame-graphs-1`, and it shows up in the page outline.
**Evidence:** Live page: `<h3 id="visualizing-the-recorded-profile-information-using-flame-graphs-1">Visualizing the recorded profile information using Flame Graphs</h3><h2 id="call-counts-analysis-with-bcc-bpf">`
**Fix:** Delete line 167.

### F5 [nit] CONFUSING: Tool prerequisites name the wrong FlameGraph script (`topics/performance-on-cpu.md:82`)
**Doc says:**
> made accessible the `difffolded.pl` and `flamegraph.pl` files, to generated the collapsed stack traces and Flame Graphs.

**Problem:** The examples use `stackcollapse-perf.pl` (line 122) and `flamegraph.pl`. They never use `difffolded.pl`.
**Fix:** "Cloned Brendan Gregg's FlameGraph repo and added it to your `PATH` so `stackcollapse-perf.pl` and `flamegraph.pl` are available."

### F6 [nit] STYLE: Typos and misspelled name (`topics/performance-on-cpu.md:18`)
**Doc says:**
> Brendan Greg

**Problem:** These errors repeat across the page.
- "Brendan Greg" should be "Brendan Gregg" (lines 18, 76, 82, 114, 228).
- "which we intent to keep" should be "intend" (line 39).
- "to generated" should be "to generate" (line 82).
- "we've remove" should be "we've removed" (line 162).
- "the number stalled cycles" should be "the number of stalled cycles" (line 204).
- "(some more suited than other" should be "others" (line 16).
**Fix:** Apply the corrections listed above.

### F7 [nit] CONFUSING: perf stat event names are Intel-specific, and the page doesn't say so (`topics/performance-on-cpu.md:207`)
**Doc says:**
> $ perf stat -e "cpu-clock,cpu-cycles,instructions,uops_executed.core,uops_executed.stall_cycles,...cycle_activity.stalls_l1d_miss"

**Problem:** `uops_executed.*` and `cycle_activity.*` are Intel PMU events. On AMD, Graviton/ARM, or most cloud VMs without PMU passthrough, they fail or show `<not supported>`. Line 199 hints at `perf list` but doesn't connect it to this command.
**Fix:** Add before the command: "The `uops_executed.*` and `cycle_activity.*` events below are Intel-specific. On other CPUs, use `perf list` to find the equivalents. On virtual machines, PMCs may not be exposed at all."

### F8 [nit] AI-NATIVE: Opening heading and first paragraph don't say what the page covers (`topics/performance-on-cpu.md:7`)
**Doc says:**
> ## Filling the performance checklist
>
> Valkey is developed with a great emphasis on performance. We do our best with every release to make sure you'll experience a very stable and fast product.

**Problem:** The first heading and first paragraph are filler. The scope (on-CPU profiling of valkey-server with perf, bcc and PMCs, for people investigating regressions) only appears at line 59, under a 15-word H2 (line 57).
**Fix:** Delete lines 9-10. Open with: "This guide shows how to find where valkey-server spends CPU time, using perf, bcc/BPF tools, and hardware performance counters. It's for engineers investigating a performance regression or optimizing Valkey." Rename line 7 to `## Confirm the CPU is the bottleneck` (merging in the 26-31 subsection), and line 57 to `## On-CPU analysis toolkit`.

### F9 [nit] STYLE: Code blocks are indented, with no fences or language tags (`topics/performance-on-cpu.md:55`)
**Doc says:**
>     $ make SERVER_CFLAGS="-g"

**Problem:** Every code block is an indented block (lines 55, 95, 105, 121-123, 141, 145, 160, 165, 175-186, 207-224). They render as `data-lang="plain"`, so commands and output can't be told apart. Lines 143-147 also split one instruction across two code blocks.
**Fix:** Convert to fenced blocks: ```` ```bash ```` for commands, ```` ```text ```` for output. Merge lines 143-147 into: "On the machine where you run `perf report`, extract the archive:" followed by a single block.

### F10 [nit] STYLE: perf-script link points to an outdated mirror (`topics/performance-on-cpu.md:126`)
**Doc says:**
> [perf script](https://linux.die.net/man/1/perf-script)

**Problem:** linux.die.net hosts years-old man pages. The other perf links on the page use man7.org.
**Fix:** Use `https://man7.org/linux/man-pages/man1/perf-script.1.html` (returns 200).

## Unverified
- topics/performance-on-cpu.md:151: "as of Linux kernel 4.9, BPF-optimized profiling is now fully available". It matches bcc's own docs for `profile`, but I didn't check it against kernel sources.
- topics/performance-on-cpu.md:175: I didn't run the bcc `funccount` and `profile` commands (no bcc or root on the review host). F2 is based only on symbols in the source.
