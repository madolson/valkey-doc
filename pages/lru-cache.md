VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=4

# Key eviction (`lru-cache`)

https://valkey.io/topics/lru-cache/

Policy names, config defaults, the LFU table and the 32-bit 3GB limit all match 9.1.2. The gaps are operational: the page never states that `noeviction` is the default or what error clients get, it describes `mem_not_counted_for_evict` incompletely, and it does not mention that replicas ignore `maxmemory` by default.

## Findings

### F1 [minor] MISSING: Default policy and OOM error not stated (`topics/lru-cache.md:29`)
**Doc says:**
> When the specified amount of memory is reached, how **eviction policies** are configured determines the default behavior.

> * **noeviction**: New values aren't saved when memory limit is reached.
**Problem:** The page never says which policy is the default. A reader who sets only `maxmemory` gets write errors, not eviction. The page also never shows the error clients receive, which is the string people search for.
**Evidence:** `valkey@9.1.2:src/config.c:3365`:
> createEnumConfig("maxmemory-policy", NULL, MODIFIABLE_CONFIG, maxmemory_policy_enum, server.maxmemory_policy, MAXMEMORY_NO_EVICTION, NULL, NULL),

Live 9.1.2:
```
$ valkey-cli config get maxmemory-policy
maxmemory-policy
noeviction
$ valkey-cli config set maxmemory 1
OK
$ valkey-cli set a b
OOM command not allowed when used memory > 'maxmemory'.
```
**Fix:** Replace line 29 with: "The default `maxmemory-policy` is `noeviction`. Setting only `maxmemory` makes write commands fail with `OOM command not allowed when used memory > 'maxmemory'.` once the limit is reached. Set an eviction policy to have Valkey remove keys instead." Change the line 51 bullet to: "**noeviction** (default): Keys are never evicted. Commands that may use more memory fail with an OOM error. Reads and deletes still work."

### F2 [minor] INACCURATE: `mem_not_counted_for_evict` contents are incomplete (`topics/lru-cache.md:98`)
**Doc says:**
> * `mem_not_counted_for_evict`: The amount of memory not counted for eviction. This includes the replication buffer and AOF buffer.

Also line 36: "Valkey needs some RAM as a buffer to send data to the replicas and AOF files. This memory is not included in the used memory count".
**Problem:** In 9.1.2 only the replication buffer above `repl-backlog-size` is excluded. The backlog itself is counted against `maxmemory`. Slot export buffers from atomic slot migration are also excluded and are not mentioned.
**Evidence:** `valkey@9.1.2:src/evict.c:218-236`:
```
if ((long long)server.repl_buffer_mem > server.repl_backlog_size) {
    ...
    size_t counted_mem = server.repl_backlog_size + extra_approx_size;
    if (server.repl_buffer_mem > counted_mem) {
        overhead += (server.repl_buffer_mem - counted_mem);
...
if (server.aof_state != AOF_OFF) {
    overhead += sdsAllocSize(server.aof_buf);
...
if (clusterIsAnySlotExporting()) {
    overhead += clusterGetTotalSlotExportBufferMemory();
```
**Fix:** Line 98: "`mem_not_counted_for_evict`: Memory excluded from the eviction check: the replica output buffers beyond `repl-backlog-size`, the AOF buffer, and slot export buffers during atomic slot migration. The replication backlog itself counts toward `maxmemory`." Line 36: replace "This memory is not included" with "Most of this memory is not included (the replication backlog is)".

### F3 [minor] MISSING: Replicas ignore `maxmemory` by default (`topics/lru-cache.md:51`)
**Doc says:**
> When a database uses replication, this applies to the primary database

and section "Considerations for `maxmemory` when using replication" (lines 34-42).
**Problem:** The clause on line 51 is unclear: every policy, not only `noeviction`, is enforced on the primary. Replicas do not evict on their own. They apply the primary's DELs, because `replica-ignore-maxmemory` defaults to `yes`. The replication section never says this, so a reader cannot tell why a replica can grow past its own `maxmemory`.
**Evidence:** Live 9.1.2:
```
$ valkey-cli config get replica-ignore-maxmemory
replica-ignore-maxmemory
yes
```
`valkey@9.1.2:valkey.conf:1386`: `# replica-ignore-maxmemory yes`
**Fix:** Delete the clause from line 51. Add to the replication section: "Eviction runs only on the primary. Replicas apply the primary's deletions and ignore their own `maxmemory` (`replica-ignore-maxmemory yes` by default). Give replicas the same or more memory than the primary, or a replica can run out of memory."

### F4 [minor] CONFUSING: Eviction sequence is misordered and omits tenacity (`topics/lru-cache.md:85`)
**Doc says:**
> * A client runs a new command, resulting in more data added.
> * Valkey checks the memory usage, and if it is greater than the `maxmemory` limit , it evicts keys according to the policy.
> * A new command is executed, and so forth.
**Problem:** Valkey checks memory and evicts *before* it executes each command, so a command can push memory over the limit and the eviction happens on the next command. Eviction is also time-bounded per command and continues in a background time event, controlled by `maxmemory-eviction-tenacity` (default 10, range 0-100). The page never mentions that setting.
**Evidence:** `valkey@9.1.2:src/server.c:4488` (in `processCommand`, before `call()`): `int out_of_memory = (performEvictions() == EVICT_FAIL);`. `valkey@9.1.2:src/evict.c:318`: "The evictionTimeProc is started when "maxmemory" has been breached and...". `valkey@9.1.2:src/config.c:3405`: `createIntConfig("maxmemory-eviction-tenacity", ..., 0, 100, server.maxmemory_eviction_tenacity, 10, ...)`.
**Fix:** Replace the list with: "1. Before running each command, Valkey compares memory usage with `maxmemory`. 2. If usage is over the limit, Valkey evicts keys according to the policy. If no key can be evicted, commands that may use more memory are rejected with an OOM error. 3. The command runs and may push usage over the limit again." Then add: "Eviction per command is time-limited. If the limit is still exceeded, Valkey keeps evicting in the background. `maxmemory-eviction-tenacity` (0-100, default 10) controls how much time is spent. Raise it if write traffic outpaces eviction."

### F5 [nit] STYLE: Grammar errors across the page (`topics/lru-cache.md:157`)
**Doc says:**
> Later more 50% of keys are added

Also line 26 "results into", 86 "`maxmemory` limit , it", 159 "three kind of dots", 166 "will instead only *probabilistically* evicts", 175 "what are good candidate", 182-183 "To experiment in production with different values ... is very simple."
**Problem:** Ungrammatical sentences.
**Fix:** 157 "Then 50% more keys are added". 26 "results in". 86 remove the space before the comma. 159 "three kinds of dots". 166 "only *probabilistically* evicts". 175 "good candidates". 182-183 "To experiment with sample sizes in production, use `CONFIG SET maxmemory-samples <count>`."

### F6 [nit] STYLE: Config name capitalized in heading; code blocks have no language tag (`topics/lru-cache.md:14`)
**Doc says:**
> ## `Maxmemory` configuration directive
**Problem:** The config name is `maxmemory`. In a code span the capital M reads as a different identifier. Indented blocks at lines 24 and 148 and fenced blocks at 102, 110, 124, 214, 223 have no language tag.
**Fix:** Heading: "## Set a memory limit with `maxmemory`". Turn the indented blocks into fenced blocks. Tag the config snippets (24, 148, 214) and the INFO output (110, 124) as `text`, and do the same for 102 and 223.

### F7 [nit] AI-NATIVE: Generic intro and weak description (`topics/lru-cache.md:6`)
**Doc says:**
> When Valkey is used as a cache, it is often convenient to let it automatically evict old data as you add new data. This behavior is well known in the developer community, since it is the default behavior for the popular *memcached* system.

Front matter: `description: Overview of Valkey key eviction policies (LRU, LFU, etc.)`
**Problem:** The memcached comparison is filler. The intro does not say who the page is for or that the default is no eviction.
**Fix:** Replace lines 6-12 with: "This page explains how to cap Valkey memory with `maxmemory`, choose a `maxmemory-policy` for when the cap is reached, monitor eviction, and tune the approximated LRU and LFU algorithms. It is for operators running Valkey as a cache. By default Valkey does not evict: with `maxmemory` set and the default `noeviction` policy, writes fail once the limit is reached." Description: "Configure maxmemory and eviction policies (LRU, LFU, random, TTL) in Valkey, monitor evictions, and tune sampling and LFU parameters."

### F8 [nit] AI-NATIVE: Defaults and ranges only shown inside snippets (`topics/lru-cache.md:148`)
**Doc says:**
> maxmemory-samples 5

and lines 214-217 `lfu-log-factor 10` / `lfu-decay-time 1`.
**Problem:** The defaults and valid ranges people want to quote are only implied by the config snippets. Ranges are never stated.
**Evidence:** `valkey@9.1.2:src/config.c:3400-3405`: `lfu-log-factor` 0..INT_MAX default 10, `lfu-decay-time` 0..INT_MAX default 1, `maxmemory-samples` 1..64 default 5, `maxmemory-eviction-tenacity` 0..100 default 10. Line 3365: `maxmemory-policy` default `noeviction`. Line 3467: `maxmemory` default 0.
**Fix:** Add a table after "Eviction policies" with columns Config / Default / Range / Purpose for `maxmemory`, `maxmemory-policy`, `maxmemory-samples`, `maxmemory-eviction-tenacity`, `lfu-log-factor`, `lfu-decay-time`, using the values above.
