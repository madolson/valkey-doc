VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=5

# Distributed Locks (`distlock`)

https://valkey.io/topics/distlock/

Algorithm content holds up for 9.1.2 (DELIFEQ since 9.0.0 and the wall-clock expiry caveat both check out). Most important fix: the "earlier versions" Lua release script uses `server.call`, which does not exist on Valkey 7.2, and the persistence advice names a config (`fsync=always`) that does not exist.

## Findings

### F1 [minor] INACCURATE: Fallback Lua script fails on Valkey 7.2 and has no invocation (`topics/distlock.md:94`)
**Doc says:**
> In earlier versions of Valkey, the same behavior can be achieved using a Lua script:
>
>     if server.call("get",KEYS[1]) == ARGV[1] then

**Problem:** The `server` Lua global was added in Valkey 8.0. On 7.2, the only "earlier version" that lacks DELIFEQ and also lacks `server.call`, the script errors. The page also never says how to run it (EVAL, number of keys, argument order).
**Evidence:** `valkey@8.0.0:src/eval.c:229`: `lua_setglobal(lua, "server");`. `git grep '"server"' 7.2.4 -- src/eval.c src/script_lua.c src/function_lua.c` returns nothing. On 9.1.2 the script works when called correctly:
```
$ valkey-cli -p 20360 EVAL 'if server.call("get",KEYS[1]) == ARGV[1] then return server.call("del",KEYS[1]) else return 0 end' 1 resource_name my_random_value
(integer) 1
```
**Fix:** Change the sentence to "In versions before 9.0.0, use a Lua script. Use `redis.call` on Valkey 7.2; `server.call` works from Valkey 8.0." and add the invocation: `EVAL "<script>" 1 resource_name my_random_value`.

### F2 [minor] INACCURATE: `fsync=always` is not a config, and AOF is off by default (`topics/distlock.md:176`)
**Doc says:**
> If we enable AOF persistence, things will improve quite a bit. ... If Valkey is configured, as by default, to fsync on disk every second ... we need to enable `fsync=always` in the persistence settings.

**Problem:** No config named `fsync` exists. The setting is `appendfsync always`, and it only applies with `appendonly yes`, which is off by default. A reader who searches for `fsync` in valkey.conf or runs `CONFIG SET fsync` finds nothing.
**Evidence:** `valkey@9.1.2:src/config.c:3366`: `createEnumConfig("appendfsync", NULL, MODIFIABLE_CONFIG, aof_fsync_enum, server.aof_fsync, AOF_FSYNC_EVERYSEC, ...)`. Live run:
```
$ valkey-cli CONFIG GET appendonly
appendonly
no
$ valkey-cli CONFIG SET fsync always
ERR Unknown option or number of arguments for CONFIG SET - 'fsync'
```
**Fix:** Line 176: "If we enable AOF persistence (`appendonly yes`, off by default), things improve quite a bit." Line 177: "With the default `appendfsync everysec`, a key can be missing after a restart. To guarantee lock safety across any kind of restart, set `appendfsync always`. See [persistence](persistence.md)."

### F3 [minor] OUTDATED: Recommends RC4 as a "safe pick" for lock tokens (`topics/distlock.md:106`)
**Doc says:**
> For example a safe pick is to seed RC4 with `/dev/urandom`, and generate a pseudo random stream from that.

**Problem:** RC4 has known keystream biases and is prohibited for TLS by RFC 7465 (2015). Calling it "safe" is stale security advice; every mainstream language ships a CSPRNG that is the obvious choice.
**Evidence:** RFC 7465, "Prohibiting RC4 Cipher Suites". The paragraph dates from the 2014 Redis page.
**Fix:** Replace lines 105-107 with: "Use at least 20 random bytes from a cryptographically secure generator (`/dev/urandom`, `secrets.token_hex()` in Python, `crypto.randomBytes()` in Node.js, or a UUIDv4). A timestamp concatenated with a client ID is not safe: it is guessable and can collide."

### F4 [minor] DUPLICATE: Single-instance lock pattern is repeated on the SET page (`topics/distlock.md:81`)
**Doc says:**
> SET resource_name my_random_value NX PX 30000 ... remove the key only if it exists and the value stored at the key is exactly the one I expect to be.

**Problem:** `commands/set.md:52-75` re-explains the same pattern in depth (random token, conditional delete, the same Lua script with `server.call`), and still presents the Lua script as the release method without mentioning DELIFEQ inline. Two copies have already drifted.
**Evidence:** `commands/set.md:62-63`: "set a non-guessable large random string, called token. Instead of releasing the lock with `DEL`, send a script that only removes the key if the value matches." `commands/set.md:69`: `if server.call("get",KEYS[1]) == ARGV[1]`.
**Fix:** Keep distlock.md "Correct Implementation with a Single Instance" canonical. Cut set.md:54-75 to two sentences: "`SET key token NX PX ttl` acquires a lock; release it with [`DELIFEQ`](delifeq.md). See [Distributed Locks](../topics/distlock.md) for the full pattern and its limits."

### F5 [nit] MISSING: "Releasing the Lock" does not say how (`topics/distlock.md:141`)
**Doc says:**
> Releasing the lock is simple, and can be performed whether or not the client believes it was able to successfully lock a given instance.

**Problem:** The section names no command. The reader has to scroll back 50 lines to find DELIFEQ.
**Fix:** "Send `DELIFEQ resource_name my_random_value` (or the Lua script on versions before 9.0.0) to every instance, whether or not the client believes it locked that instance."

### F6 [nit] BROKEN: Implementation list has redirected and stale links (`topics/distlock.md:31`)
**Doc says:**
> [RedisMutex](https://github.com/malkusch/lock#redismutex) ... [Redisson](https://github.com/mrniko/redisson)

**Problem:** Links redirect to renamed repos. `dl.acm.org/citation.cfm?id=74870` (line 131) uses the legacy ACM URL format.
**Evidence:**
```
200 https://github.com/php-lock/lock <- https://github.com/malkusch/lock
200 https://github.com/redisson/redisson <- https://github.com/mrniko/redisson
403 https://dl.acm.org/citation.cfm?id=74870
```
**Fix:** Use `https://github.com/php-lock/lock`, `https://github.com/redisson/redisson`, `https://dl.acm.org/doi/10.1145/74850.74870`. Prune unmaintained libraries per the tracked issue.
Tracked: valkey-io/valkey-doc#101

### F7 [nit] STYLE: Code blocks are indented with no language tag (`topics/distlock.md:83`)
**Doc says:**
>         SET resource_name my_random_value NX PX 30000

**Problem:** Lines 83, 92, and 96-100 are indented blocks. The live page renders them as `data-lang="plain"`, so there is no Lua highlighting.
**Fix:** Use fenced blocks: ```` ```text ```` for the SET and DELIFEQ commands, ```` ```lua ```` for the script.

### F8 [nit] STYLE: Typos and stray markup (`topics/distlock.md:218`)
**Doc says:**
> Martin Kleppman's article (218); don´t (223); participates to (181); approximately at the same rate (127); `## Analysis of Redlock` followed by `---` (235)

**Problem:** The name is misspelled (line 237 has it right: Kleppmann). Line 223 uses an acute accent where an apostrophe belongs. Line 235 renders a stray horizontal rule under the heading.
**Fix:** "Kleppmann's", "don't", "participates in", "at approximately the same rate". Delete line 235.

### F9 [nit] AI-NATIVE: Weak description, misleading heading, filler (`topics/distlock.md:4`)
**Doc says:**
> description: A distributed lock pattern with Valkey (4); ### Is the Algorithm Asynchronous? (125); ## Want to help? ... Thanks in advance! (228-232); We hope that the community will analyze it, provide feedback (18-20)

**Problem:** The description names no algorithm or audience. The section under "Is the Algorithm Asynchronous?" covers the clock-drift assumption and the real validity bound, so the heading hides it. "Want to help?" and the 2014-era call for feedback carry no information.
**Fix:** Description: "How to implement locks on a single Valkey instance with SET NX and DELIFEQ, and across independent primaries with the Redlock algorithm, including its safety limits." Rename line 125 to "### Clock Drift Assumptions". Delete lines 18-20 ("We hope ... designs.") and lines 228-232.

## Unverified
- topics/distlock.md:27-47: whether each listed library is maintained or supports Valkey. Only checked HTTP status; all GitHub links return 200.
