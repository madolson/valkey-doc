VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=4

# Keyspace (`keyspace`)

https://valkey.io/topics/keyspace/

The page is accurate in substance, but the Scan and Keys sections are copied from the command pages, and the page links to almost nothing: the live page has one in-content link. The most important fix: replace the copied SCAN/KEYS text with links to commands/scan.md and commands/keys.md, and link every command the page names.

## Findings

### F1 [minor] INACCURATE: 512 MB key size is a configurable default, not a fixed maximum (`topics/keyspace.md:27`)
**Doc says:**
> The maximum allowed key size is 512 MB.

**Problem:** The 512 MB limit comes from `proto-max-bulk-len`, which is runtime-modifiable up to LONG_MAX. Readers will think the limit is hard-coded.
**Evidence:** valkey@9.1.2:src/config.c:3461
> `createLongLongConfig("proto-max-bulk-len", NULL, DEBUG_CONFIG | MODIFIABLE_CONFIG, 1024 * 1024, LONG_MAX, server.proto_max_bulk_len, 512ll * 1024 * 1024, ...`

valkey@9.1.2:src/networking.c:3812 rejects bulk args `ll > server.proto_max_bulk_len`. Live 9.1.2:
```
CONFIG GET proto-max-bulk-len  -> 536870912
CONFIG SET proto-max-bulk-len 2gb -> OK
CONFIG GET proto-max-bulk-len  -> 2147483648
```
**Fix:** "By default the maximum key size is 512 MB, set by the `proto-max-bulk-len` configuration. Keys anywhere near this size are a bad idea (see above)."
Tracked: valkey-io/valkey-doc#97

### F2 [minor] DUPLICATE: Scan and Keys sections are copied from the command pages (`topics/keyspace.md:111`)
**Doc says:**
> Since `SCAN` allows for incremental iteration, returning only a small number of elements per call, it can be used in production without the downside of commands like `KEYS` or `SMEMBERS` ...

and lines 120-140 (the KEYS warning, "consider `SCAN` or [sets][tdts]", and the glob pattern list).
**Problem:** Lines 111-114 are copied from commands/scan.md:8-10. Lines 120-140 are copied word for word from commands/keys.md:8-29, including the stray comma in `` `hallo,` ``. Two copies will drift. The command pages are the right canonical source.
**Fix:** Replace lines 108-140 with: "Use [`SCAN`](../commands/scan.md) to iterate the keyspace incrementally. Each call returns a few keys, so it is safe in production, but keys added or removed during the iteration may or may not be returned. [`KEYS`](../commands/keys.md) returns every key matching a glob pattern in one call and blocks the server until it finishes. Use it only for debugging. The pattern syntax is documented on the `KEYS` page."

### F3 [minor] CONFUSING: SMEMBERS called a "blocking command", and the sentence is a fragment (`topics/keyspace.md:113`)
**Doc says:**
> However while blocking commands like `SMEMBERS` are able to provide all the elements that are part of a Set in a given moment.
> The `SCAN` family of commands only offer limited guarantees ...

**Problem:** In Valkey, "blocking command" means BLPOP, BLMOVE and similar commands that wait for data. SMEMBERS is not one. The sentence was split from commands/scan.md:10, so line 113 has no main clause. It also drifts from keys to Set members on a page about the keyspace.
**Fix:** If F2 is not applied: "Unlike `KEYS`, which returns every matching key in one atomic call, `SCAN` only offers limited guarantees, because the keyspace can change during the iteration."

### F4 [minor] MISSING: no links to command pages or to the expiration details (`topics/keyspace.md:35`)
**Doc says:**
> For example the `EXISTS` command ... `DEL` ... `TYPE` ... `EXPIRE` ... `PERSIST` ... `TTL` ... `PEXPIRE` ... `PTTL` ... the full list of `SET` options.

**Problem:** None of the eleven commands is linked. The live page has one in-content link (`/topics/data-types#sets`). Line 104 sends readers to "the full list of `SET` options" with no link. The expiration section skips how expired keys are reclaimed (lazy vs. active), and how expires are replicated. commands/expire.md:106-172 covers this ("How Valkey reclaims expired keys", "How expires are handled in the replication link and AOF file"), but this page never links it.
**Evidence:** `curl -sL https://valkey.io/topics/keyspace/ | grep -o '<a href=...'` returns only the breadcrumb, `/topics/data-types#sets`, and footer links.
**Fix:** Link each command on first mention, for example [`EXISTS`](../commands/exists.md) and [`SET`](../commands/set.md). Add after line 73: "For how Valkey reclaims expired keys and replicates expirations, see [Valkey expires](../commands/expire.md#appendix-valkey-expires)."

### F5 [nit] EXAMPLE: expire example is not runnable, and the TTL output does not match an immediate run (`topics/keyspace.md:81`)
**Doc says:**
> `> get key (immediately)`
>
> `> ttl key` / `(integer) 9`

**Problem:** If a reader pastes the annotations, they get an error. Run right after `SET ... EX 10`, `TTL` returns 10, not 9. The pattern repeats: every example (lines 39, 56, 77, 94) is an indented block with no language tag.
**Evidence:** 9.1.2:
```
$ valkey-cli -p 20620 get key '(immediately)'
ERR wrong number of arguments for 'get' command
$ valkey-cli set key 100 ex 10 ; valkey-cli ttl key
OK
10
```
**Fix:** Move the annotations into prose ("Immediately: `GET key` returns `"some-value"`. After 5 seconds: `(nil)`"). Show `(integer) 10`, or say "(a few seconds later)". Convert all four blocks to fenced blocks.

### F6 [nit] AI-NATIVE: opening states no scope, and leftover tutorial phrasing (`topics/keyspace.md:7`)
**Doc says:**
> Valkey keys are binary safe; ...

> Before moving on, we should look at an important Valkey feature ...

**Problem:** The first paragraph does not say the page covers key naming, generic key commands, expiration, and iteration. "Before moving on" (line 67) and "Another way to iterate" are tutorial carry-overs with nothing to move on from.
**Fix:** Open with: "This page covers how Valkey keys work: naming rules and limits, commands that work on keys of any type, key expiration, and iterating the keyspace with `SCAN` and `KEYS`." Line 67: "Key expiration lets you set a timeout, or time to live (TTL), on a key of any type. When the TTL elapses, Valkey deletes the key."

### F7 [nit] CONFUSING: persistence sentence is ungrammatical and vague (`topics/keyspace.md:73`)
**Doc says:**
> Information about expires are replicated and persisted on disk, the time virtually passes when your Valkey server remains stopped (this means that Valkey saves the date at which a key will expire).

**Fix:** "Expirations are replicated and persisted to disk as absolute Unix timestamps, so time keeps counting while the server is stopped. A key whose expiration passed during downtime is gone after a restart."

### F8 [nit] STYLE: typos (`topics/keyspace.md:109`)
**Doc says:**
> To incrementally  iterate

> `h[ae]llo` matches `hello` and `hallo,` but not `hillo` (line 136)

**Fix:** Change "incrementally  iterate" (double space) to "incrementally iterate". Change `` `hallo,` `` to `` `hallo`, `` (also in commands/keys.md:23).
