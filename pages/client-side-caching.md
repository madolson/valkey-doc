VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=6 nit=4

# Client-side caching (`client-side-caching`)

https://valkey.io/topics/client-side-caching/

The protocol examples are accurate on 9.1.2. But the page has find-and-replace damage from the Redis rename ("Valkey numbers", "Valkey 2"), still describes the removed "caching slots" design, and never names the config that limits server memory. Fix L116 first. As written it makes no sense.

## Findings

### F1 [minor] INACCURATE: "Valkey numbers" should be "database numbers" (`topics/client-side-caching.md:116`)
**Doc says:**
> There is a single keys namespace, not divided by Valkey numbers. So if a client is caching the key `foo` in Valkey 2, and some other client changes the value of the key `foo` in Valkey 3, an invalidation message will still be sent. This way we can ignore Valkey numbers

**Problem:** A rename sed turned "database" into "Valkey". The sentence now reads as if it were about Valkey versions or servers. The behavior it describes (invalidation ignores the DB number) is real.
**Evidence:** Live 9.1.2 run. The data connection did `SELECT 2`, `GET foo`. A second client did `SELECT 3`, `SET foo bar`. The redirect connection received:
```
*3\r\n$7\r\nmessage\r\n$20\r\n__redis__:invalidate\r\n*1\r\n$3\r\nfoo\r\n
```
**Fix:** "There is a single key namespace, not divided by database number. If a client caches key `foo` in database 2 and another client modifies `foo` in database 3, an invalidation message is still sent. Ignoring database numbers reduces memory use and implementation complexity."

### F2 [minor] OUTDATED: "caching slots" describe a design that no longer exists (`topics/client-side-caching.md:115`, `:174`, `:207`)
**Doc says:**
> the information will be incrementally garbage collected as caching slots are invalidated.

> The client will check if there are cached keys in this caching slot, and will evict the information that is no longer valid.

> caching slots will get invalidated during the course of the time.

**Problem:** "Caching slots" are never defined on the page. They come from an early design that hashed keys into slots. The current tracking table is keyed by the key name, and the invalidation message carries the key name itself (L172 shows `foo`). L174 contradicts the example right above it.
**Evidence:** valkey@9.1.2:src/tracking.c:32-34:
```
/* The tracking table is constituted by a radix tree of keys, each pointing
 * to a radix tree of client IDs, used to track the clients that may have
 * certain keys in their local, client side, cache.
```
**Fix:** L115: "...the information is garbage collected incrementally as the keys are invalidated." L174: "The client evicts `foo` from its local cache." L207: "...cached keys will get invalidated over time."

### F3 [minor] MISSING: `tracking-table-max-keys` is never named, and its default is not stated (`topics/client-side-caching.md:114`, `:330`)
**Doc says:**
> The invalidation table can contain a maximum number of entries.

> Be sure to configure a suitable value for the maximum number of keys remembered by Valkey

**Problem:** The page tells operators to configure a limit but never says which config sets it, what the default is, or how to see current usage. This is the main operational fact on the page.
**Evidence:** valkey@9.1.2:src/config.c:3481:
```
createSizeTConfig("tracking-table-max-keys", NULL, MODIFIABLE_CONFIG, 0, LONG_MAX, server.tracking_table_max_keys, 1000000, ...
```
valkey@9.1.2:src/server.c:6481-6483 shows the INFO fields `tracking_total_keys`, `tracking_total_items`, `tracking_total_prefixes`. Line 6222 shows `tracking_clients`.
**Fix:** At L330: "Set the limit with `tracking-table-max-keys` (default 1000000, modifiable at runtime, 0 means no limit). When the table is over the limit, the server invalidates keys to get back under it, and clients holding those keys receive invalidation messages. Monitor usage with the `tracking_total_keys` and `tracking_total_items` fields in `INFO stats` and `tracking_clients` in `INFO clients`." Check the 0 semantics against valkey.conf before publishing.

### F4 [minor] INACCURATE: the prefix overlap rule applies per client, not globally (`topics/client-side-caching.md:251`)
**Doc says:**
> No two prefixes can track overlapping parts of the keyspace. For instance, having the prefix "foo" and "foob" would not be allowed

**Problem:** Overlap is rejected only within one client's prefix set. Different clients can register `foo` and `foob`. The page also doesn't say that an overlap returns an error.
**Evidence:** 9.1.2 run:
```
client A: CLIENT TRACKING on BCAST PREFIX foo PREFIX foob
-ERR Prefix 'foo' overlaps with another provided prefix 'foob'. Prefixes for a single client must not overlap.
client A: CLIENT TRACKING on BCAST PREFIX foo   -> +OK
client B: CLIENT TRACKING on BCAST PREFIX foob  -> +OK
```
**Fix:** "A single client cannot register overlapping prefixes. For example, `PREFIX foo PREFIX foob` returns an error because both would match `foobar`. Use `foo` alone. Different clients may register overlapping prefixes."

### F5 [minor] MISSING: no links to the command reference, and OPTOUT, `CLIENT CACHING no` and redirect failure are absent (`topics/client-side-caching.md:225`)
**Doc says:**
> In order to do this, tracking must be enabled using the OPTIN option:

**Problem:** The page has no links at all. It covers OPTIN but never mentions OPTOUT or `CLIENT CACHING no`. It also skips what happens when a redirect target is missing or disconnects. A client implementer needs all of these.
**Evidence:** commands/client-tracking.md:24 documents `tracking-redir-broken` push messages. Line 28 documents `OPTOUT`. A 9.1.2 run of `CLIENT TRACKING on REDIRECT 99999` returned `-ERR The client ID you want redirect to does not exist`.
**Fix:** Link [`CLIENT TRACKING`](../commands/client-tracking.md), [`CLIENT CACHING`](../commands/client-caching.md), [`CLIENT TRACKINGINFO`](../commands/client-trackinginfo.md) and [`CLIENT GETREDIR`](../commands/client-getredir.md) at L86. After L239 add: "The `OPTOUT` option does the reverse: keys are tracked by default, and `CLIENT CACHING no` excludes the next command." In the Two connections section add: "If the redirect target disconnects, RESP3 clients receive a `tracking-redir-broken` push message."

### F6 [minor] INACCURATE: broadcasting mode is said to use no server memory (`topics/client-side-caching.md:89`, `:246`, `:330`)
**Doc says:**
> so this mode costs no memory at all in the server side

> does not consume any memory on the server side

> use the BCAST mode that consumes no memory at all on the Valkey side

**Problem:** BCAST still keeps a prefix table with per-prefix client sets. It also buffers every key modified in the current event loop cycle under each matching prefix. Memory is not per-key-read, but it is not zero.
**Evidence:** valkey@9.1.2:src/tracking.c:52-56:
```
typedef struct bcastState {
    rax *keys;    /* Keys modified in the current event loop cycle. */
    rax *clients; /* Clients subscribed to the notification events for this
                     prefix. */
```
**Fix:** Replace with "uses no per-key memory on the server side; memory is proportional to the number of prefixes and subscribed clients."

### F7 [nit] STYLE: "the Valkey" / "a Valkey" rename artifacts (`topics/client-side-caching.md:13`, `:24`, `:40`, `:75`)
**Doc says:**
> the application servers ask the Valkey about such information

> without contacting the Valkey again

> a networked service like a Valkey

> costing the Valkey more CPU time

**Problem:** This is the same sed artifact as F1. It reads as broken English.
**Fix:** "ask Valkey", "contacting Valkey again", "a networked service like Valkey", "costing Valkey more CPU time".

### F8 [nit] AI-NATIVE: the heading and intro don't say what the page covers (`topics/client-side-caching.md:7`, `:57`, `:91`)
**Doc says:**
> ## There are two hard problems in computer science...

> To recap, for now let's forget for a moment about the broadcasting mode

**Problem:** The joke heading doesn't say the section is about cache invalidation. The first paragraph never says the page documents the `CLIENT TRACKING` feature or that it is aimed at client library authors. L91 says "To recap" but no recap follows.
**Fix:** Retitle L57 "The cache invalidation problem". Add to L7: "This page explains Valkey's server-assisted client-side caching (`CLIENT TRACKING`): default, opt-in and broadcasting modes, the RESP2 and RESP3 invalidation protocols, and implementation advice for client libraries." Replace L91-92 with "The rest of this section describes the default mode. Broadcasting is described in [Broadcasting mode](#broadcasting-mode)."

### F9 [nit] EXAMPLE: code blocks have no language tag, and the output styles are mixed (`topics/client-side-caching.md:126`, `:141`, `:155`, `:163`, `:227`, `:231`)
**Doc says:**
> ```
> (Connection 1 -- used for invalidations)

>     CLIENT CACHING YES
>     +OK
>     GET foo
>     "bar"

**Problem:** The fenced blocks have no language. L227-234 and L278-293 are indented blocks. L231-234 mixes raw RESP (`+OK`) with valkey-cli-style output (`"bar"`) in one block.
**Fix:** Fence them all with a tag (`text` for raw RESP and the sequence diagrams). In L234 change `"bar"` to `$3` / `bar` to match the RESP2 blocks above it.

### F10 [nit] CONFUSING: Two connections mode is one dense paragraph (`topics/client-side-caching.md:120`)
**Doc says:**
> Using the new version of the Valkey protocol, RESP3, it is possible to run the data queries and receive the invalidation messages in the same connection. However many client implementations may prefer ...

**Problem:** One paragraph mixes three facts: RESP3 can use one connection, `REDIRECT` sends invalidations to another connection, and RESP2 must use redirect. Calling RESP3 "new" (L120) and RESP2 "old" (L122) is dated.
**Fix:** Split it into a short list: "RESP3: invalidations can arrive as push messages on the data connection. RESP2: you must use `REDIRECT <client-id>` to a connection subscribed to `__redis__:invalidate`. Several data connections can redirect to one invalidation connection (useful for pools)." Drop "new" and "old".
