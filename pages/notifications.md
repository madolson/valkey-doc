VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=7 nit=5

# Keyspace notifications (`notifications`)

https://valkey.io/topics/notifications/

The event catalog has several wrong event names and orderings when you check it against a live 9.1.2 server, and it is missing events added since the page was written. Fix the event list first: `zremrangebyscore`/`zremrangebyrank` names, the reversed LMOVE/RPOPLPUSH ordering guarantee, and `EXPIRE` with a past time emitting `expired`, not `del`.

## Findings

### F1 [minor] INACCURATE: Key-event channel described as per-key (`topics/notifications.md:31`)
**Doc says:**
> The first channel listens to all the events targeting the key `mykey` and the other channel listens only to `del` operation events on the key `mykey`
**Problem:** `__keyevent@0__:del` receives `del` events for every key in db 0, not only `mykey`. The key name is the message payload, as line 43 says. This contradicts line 43 and the event-type model.
**Evidence:** valkey@9.1.2:src/notify.c: the keyevent channel is built as `__keyevent@<db>__:<event>` with the key as the message. Live: `__keyevent@0__:del` delivered `"z"`, `"a"`, `"hx"`, `"s"` from different keys.
**Fix:** "The first channel receives every event that targets the key `mykey`. The second channel receives every `del` event in database 0, whichever key it targets."

### F2 [minor] INACCURATE: Wrong event names for ZREMRANGEBYSCORE/ZREMRANGEBYRANK, ZREMRANGEBYLEX missing (`topics/notifications.md:126`)
**Doc says:**
> `ZREMBYSCORE` generates a single `zrembyscore` event. ... `ZREMBYRANK` generates a single `zrembyrank` event.
**Problem:** The commands are named ZREMRANGEBYSCORE and ZREMRANGEBYRANK. Their events are `zremrangebyscore` and `zremrangebyrank`. Anyone subscribing to `__keyevent@0__:zrembyscore` receives nothing. ZREMRANGEBYLEX (`zremrangebylex`) is not listed.
**Evidence:** valkey@9.1.2:src/t_zset.c:1995-2006:
```
        notify_type = "zremrangebyrank";
        notify_type = "zremrangebyscore";
        notify_type = "zremrangebylex";
```
Live 9.1.2:
```
"__keyevent@0__:zremrangebyscore","z"
"__keyevent@0__:zremrangebyrank","z"
```
**Fix:** Change both lines to "`ZREMRANGEBYSCORE`, `ZREMRANGEBYRANK` and `ZREMRANGEBYLEX` generate a single `zremrangebyscore`, `zremrangebyrank` or `zremrangebylex` event respectively. When the resulting sorted set is empty and the key is removed, an additional `del` event is generated." This also fixes "the key is generated" (should be "removed"), which appears at lines 125, 126 and 127.

### F3 [minor] INACCURATE: LMOVE/RPOPLPUSH push event comes before the pop event, not after (`topics/notifications.md:109`)
**Doc says:**
> In both cases the order is guaranteed (the `lpush` event will always be delivered after the `rpop` event).

and line 110: "the `lpush`/`rpush` event will always be delivered after the `lpop`/`rpop` event".
**Problem:** The page promises the wrong order. The server emits the destination push event (and `new`, if the destination did not exist) first, then the source pop event. A consumer that relies on the documented order will see events in the opposite sequence. This is the same on unstable.
**Evidence:** valkey@9.1.2:src/t_list.c:1116-1119 (same on origin/unstable):
```
    value = listTypePop(sobj, wherefrom);
    serverAssert(value); /* assertion for valgrind (avoid NPD) */
    lmoveHandlePush(c, c->argv[2], dobj, value, whereto);   // emits lpush/rpush
    listElementsRemoved(c, touchedkey, wherefrom, sobj, 1, NULL); // emits lpop/rpop, del
```
Live `RPOPLPUSH l l2`:
```
"__keyevent@0__:new","l2"
"__keyevent@0__:lpush","l2"
"__keyevent@0__:rpop","l"
```
**Fix:** "...In both cases the order is guaranteed: the `lpush`/`rpush` event on the destination is delivered before the `lpop`/`rpop` event on the source." Apply to both lines 109 and 110.

### F4 [minor] INACCURATE: EXPIRE with a past time emits `expired`, not `del` (`topics/notifications.md:93`)
**Doc says:**
> Note that when these commands are called with a negative timeout value or timestamp in the past, the key is deleted and only a `del` event is generated instead.
**Problem:** On Valkey this path emits `expired` (class `x`), not `del` (class `g`). A subscriber filtering with `Kg` or on `__keyevent@*__:del` misses these deletions.
**Evidence:** valkey@9.1.2:src/expire.c:853 calls `deleteExpiredKeyFromOverwriteAndPropagate(c, key);`, and valkey@9.1.2:src/db.c:1981 does `notifyKeyspaceEvent(NOTIFY_EXPIRED, "expired", keyobj, c->db->id);`. Live `EXPIRE k -1` and `PEXPIREAT k 1`:
```
"__keyevent@0__:expired","k"
```
No `del` event. Introduced by valkey-io/valkey#865 ("Fast path in SET if the expiration time is expired").
**Fix:** "...the key is deleted and only an `expired` event is generated instead." Also check whether `SET ... EX`/`PXAT` with a past time needs the same note.

### F5 [minor] INACCURATE: HSETEX with a past expiration emits more than "a single `hexpired` event" (`topics/notifications.md:115`)
**Doc says:**
> `HSETEX` and `HGETEX` ... when these commands are called with a negative timeout value or timestamp in the past, only a single `hexpired` event is generated
**Problem:** HSETEX with a past expiration emits `hset`, `hexpire` and `hexpired`, then `del` if the hash is now empty. "Only a single `hexpired`" is wrong for HSETEX. Line 111 also lists HSETEX as emitting `hset`, so the page contradicts itself.
**Evidence:** Live `HSETEX hy EX 0 FIELDS 1 f v` on 9.1.2:
```
"__keyevent@0__:new","hy"
"__keyevent@0__:hset","hy"
"__keyevent@0__:hexpire","hy"
"__keyevent@0__:hexpired","hy"
"__keyevent@0__:del","hy"
```
**Fix:** Split HSETEX from HGETEX: "`HSETEX` generates an `hset` event and an `hexpire` event. If the expiration is in the past, it also generates an `hexpired` event, and a `del` event if the hash is left empty." Keep the "only a single `hexpired`" wording for `HGETEX` only after you verify it.

### F6 [minor] MISSING: Event list omits many commands and the `keymiss` event name (`topics/notifications.md:85`)
**Doc says:**
> Different commands generate different kind of events according to the following list.
**Problem:** The list reads as complete but leaves out events that 9.1.2 emits, so readers cannot work out the channel names. The `m` class (line 71) never names its event.
**Evidence:** `git grep notifyKeyspaceEvent 9.1.2 -- src/` shows event names that are missing from the page: `setbit` (SETBIT, BITFIELD, BITOP; src/bitops.c:725,1418), `pfadd` (PFADD, PFMERGE; src/hyperloglog.c:1692,1868), `georadiusstore`/`geosearchstore` (src/geo.c:852), `zrangestore` (src/t_zset.c:3052), `zpopmin`/`zpopmax` (ZPOPMIN, ZPOPMAX, BZPOP*, ZMPOP; src/t_zset.c:3988), `keymiss` (src/db.c:120). Live: `GET nokey` produced `"__keyevent@0__:keymiss","nokey"`. LMPOP, BLPOP, GETDEL and GETEX are also not listed.
**Fix:** Add bullets: "`SETBIT`, `BITFIELD` and `BITOP` generate a `setbit` event.", "`PFADD` and `PFMERGE` generate a `pfadd` event.", "`GEOADD` generates a `zadd` event. `GEORADIUS`/`GEOSEARCHSTORE` with STORE generate `georadiusstore`/`geosearchstore`.", "`ZRANGESTORE` generates a `zrangestore` event.", "`ZPOPMIN`, `ZPOPMAX`, `BZPOPMIN`, `BZPOPMAX` and `ZMPOP` generate `zpopmin`/`zpopmax` events, plus `del` if the key is removed.", "`LMPOP`, `BLPOP` and `BRPOP` generate `lpop`/`rpop` events.", "When a command reads a key that does not exist, a `keymiss` event is generated (class `m`)." Change line 71 to "m  Key miss events, published as `keymiss` (...)".

### F7 [minor] CONFUSING: CONFIG GET hides `n` when `A` is set (`topics/notifications.md:72`)
**Doc says:**
> n     New key events (Note: not included in the 'A' class)
**Problem:** On 9.1.2, after `CONFIG SET notify-keyspace-events KEAn`, `CONFIG GET` returns `AKE`, but `new` events are still delivered. A reader who checks their config will think `n` was ignored. This is a server rendering bug (src/notify.c `keyspaceEventsFlagsToString` emits `n` only in the non-`A` branch), and the page should warn about it until the bug is fixed.
**Evidence:**
```
$ valkey-cli -p 20940 config set notify-keyspace-events KEAn
OK
$ valkey-cli -p 20940 config get notify-keyspace-events
notify-keyspace-events
AKE
```
`new` events were still published afterwards (`"__keyevent@0__:new","z"`).
**Fix:** Add after the table: "Note: when `A` is set, `CONFIG GET notify-keyspace-events` does not show `n` even if it is enabled." Also file a server bug against valkey-io/valkey.

### F8 [nit] INACCURATE: `ZINCR` is not a command (`topics/notifications.md:123`)
**Doc says:**
> `ZINCR` generates a `zincr` event.
**Problem:** The commands are `ZINCRBY` and `ZADD ... INCR`.
**Evidence:** valkey@9.1.2:src/t_zset.c:1922 `notifyKeyspaceEvent(NOTIFY_ZSET, incr ? "zincr" : "zadd", key, c->db->id);`. Live `ZINCRBY z 1 c` produced `zincr`.
**Fix:** "`ZINCRBY` and `ZADD` with `INCR` generate a `zincr` event."

### F9 [nit] STYLE: Typos and wrong terms in the event list (`topics/notifications.md:114`)
**Doc says:**
> (`HPEXPIRE`, `HPEXPIRE`, `HEXPIREAT` and `HPEXPIREAT`)

line 129: "when used with the `MAXLEN` subcommand"; line 51: "because while not very sensible the feature uses some CPU power"; line 75: "At least `K` or `E` should be present".
**Problem:** HPEXPIRE is listed twice. MAXLEN is an option, not a subcommand, and MINID also triggers `xtrim`. "Sensible" is meant as "expensive". "Should" understates a hard requirement.
**Fix:** "(`HPEXPIRE`, `HEXPIREAT` and `HPEXPIREAT`)"; "possibly followed by an `xtrim` event when used with the `MAXLEN` or `MINID` option"; "By default keyspace notifications are disabled because the feature uses some CPU."; "At least `K` or `E` must be present".

### F10 [nit] STYLE: Section headings skip from h1 to h3 (`topics/notifications.md:20`)
**Doc says:**
> ### Type of events
**Problem:** All five sections are `###`, and the page title is the only h1. The live page renders `<h1>` then `<h3 id="type-of-events">`.
**Fix:** Change every `###` to `##` (lines 20, 48, 83, 160, 176).

### F11 [nit] STYLE: Indented code blocks have no language tag (`topics/notifications.md:28`)
**Doc says:**
>     PUBLISH __keyspace@0__:mykey del
**Problem:** Lines 28-29, 59-73, 148-151 and 156-158 are indented blocks, so they get no language and no highlighting.
**Fix:** Convert them to fenced blocks: ```` ```text ```` for the flag table (59-73) and the output (156-158), and ```` ```sh ```` for the CLI session (148-151).

### F12 [nit] MISSING: No cross-links to Pub/Sub, PSUBSCRIBE or CONFIG SET (`topics/notifications.md:7`)
**Doc says:**
> Keyspace notifications allow clients to subscribe to Pub/Sub channels
**Problem:** The page has no links. The *fire and forget* caveat (line 16) and the cluster note (line 178) depend on Pub/Sub semantics, which are explained in topics/pubsub.md.
**Fix:** Link "Pub/Sub" to `pubsub.md`, link **CONFIG SET** (line 52) to `../commands/config-set.md`, and link `psubscribe` (line 149) to `../commands/psubscribe.md`.

## Unverified
- topics/notifications.md:116: HGETEX PERSIST emits `hpersist`. My test field had no TTL, so no event fired. Source valkey@9.1.2:src/t_hash.c:1752 suggests it does.
- topics/notifications.md:115: HGETEX with a past expiration emits only `hexpired` plus `del`. Not run.
