VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=5 nit=4

# Pub/Sub (`pubsub`)

https://valkey.io/topics/pubsub/

The page predates sharded Pub/Sub and was never fully updated for it: the subscription-count and exit-state rules are wrong once a client holds shard subscriptions, and sharded message types are undocumented. The valkey-cli limitation it describes no longer exists. Fix the subscription-count semantics first.

## Findings

### F1 [minor] INACCURATE: Count reaching zero does not mean the client left Pub/Sub state (`topics/pubsub.md:61`)
**Doc says:**
> When the last argument is zero, we are no longer subscribed to any channel, and the client can issue any kind of Valkey command as we are outside the Pub/Sub state.

and line 171-173:
> This number is the total number of channels and patterns the client is still subscribed to. So the client will exit the Pub/Sub state only when this count drops to zero

**Problem:** The count in `subscribe`/`unsubscribe`/`psubscribe`/`punsubscribe` replies covers channels and patterns only. Shard channel subscriptions are counted separately in `ssubscribe`/`sunsubscribe` replies. A RESP2 client can see `:0` and still be blocked from normal commands.
**Evidence:** valkey@9.1.2:src/pubsub.c:221-241: `clientSubscriptionsCount` vs `clientShardSubscriptionsCount`, and exit uses `clientTotalPubSubSubscriptionCount(c) == 0` (line 603, 636). Live 9.1.2 run:
```
SUBSCRIBE foo => subscribe foo :1
SSUBSCRIBE bar => ssubscribe bar :1
PSUBSCRIBE f* => psubscribe f* :2
UNSUBSCRIBE => unsubscribe foo :1
PUNSUBSCRIBE => punsubscribe f* :0
GET x => -ERR Can't execute 'get': only (P|S)SUBSCRIBE / (P|S)UNSUBSCRIBE / PING / QUIT / RESET are allowed in this context
SUNSUBSCRIBE => sunsubscribe bar :0
GET x => $-1
```
**Fix:** Line 61: "When the count is zero and the client has no shard channel subscriptions, it leaves the Pub/Sub state and can issue any command." Line 172-173: "This number is the total number of channels and patterns the client is still subscribed to. Shard channel subscriptions are counted separately in `ssubscribe` and `sunsubscribe` replies. The client exits the Pub/Sub state only when both counts reach zero."

### F2 [minor] OUTDATED: valkey-cli can send commands in subscribed mode (`topics/pubsub.md:38`)
**Doc says:**
> Please note that when using `valkey-cli`, in subscribed mode commands such as `UNSUBSCRIBE` and `PUNSUBSCRIBE` cannot be used because `valkey-cli` will not accept any commands and can only quit the mode with `Ctrl-C`.

**Problem:** Wrong since redis-cli 7.2 (inherited by all Valkey releases). valkey-cli accepts commands in subscribed mode and tracks subscription counts to leave it.
**Evidence:** valkey@9.1.2:src/valkey-cli.c:2303:
```
printf("Reading messages... (press Ctrl-C to quit or any key to type command)\r");
```
Lines 2257-2261 toggle `pubsub_mode` from `pubsub_unsharded_count + pubsub_sharded_count`; line 338 shows `(subscribed mode)` in the prompt. Introduced by bbf364a44 "redis-cli: Accept commands in subscribed mode (#11873)".
**Fix:** Replace with: "In `valkey-cli`, press any key while it is reading messages to type a command such as `UNSUBSCRIBE`. The prompt shows `(subscribed mode)` until all subscriptions are gone. Press `Ctrl-C` to quit."

### F3 [minor] MISSING: Sharded message types not documented (`topics/pubsub.md:50`)
**Doc says:** "Format of pushed messages" lists `subscribe`, `unsubscribe`, `message`; line 155 adds `psubscribe`/`punsubscribe`/`pmessage`.
**Problem:** `ssubscribe`, `sunsubscribe`, and `smessage` are never described, though sharded Pub/Sub is on the page and SSUBSCRIBE/SUNSUBSCRIBE are in the allowed-command list. Client authors reading "Client library implementation hints" will not know to handle them.
**Evidence:** Live 9.1.2: `SSUBSCRIBE bar => *3 $10 ssubscribe $3 bar :1`; `SUNSUBSCRIBE => *3 $12 sunsubscribe ...`. valkey@9.1.2:src/pubsub.c:92 uses `clientShardSubscriptionsCount` for the shard type.
**Fix:** Add to the Sharded Pub/Sub section: "Shard subscriptions use message types `ssubscribe`, `sunsubscribe`, and `smessage`, in the same three-element format as `subscribe`, `unsubscribe`, and `message`. The count in `ssubscribe`/`sunsubscribe` replies is the number of shard channels the client is subscribed to."

### F4 [minor] MISSING: Slow subscribers are disconnected by the pubsub output buffer limit (`topics/pubsub.md:45`)
**Doc says:**
> If the subscriber is unable to handle the message (for example, due to an error or a network disconnect) the message is forever lost.

**Problem:** The most common Valkey-specific loss mode is missing: a subscriber that reads too slowly is disconnected when its output buffer exceeds `client-output-buffer-limit pubsub`. Operators need the default to size consumers.
**Evidence:** Live 9.1.2 `CONFIG GET client-output-buffer-limit`:
```
normal 0 0 0 slave 268435456 67108864 60 pubsub 33554432 8388608 60
```
**Fix:** Append: "Valkey also disconnects a subscriber whose output buffer grows past `client-output-buffer-limit pubsub` (default `32mb 8mb 60`: hard limit 32 MB, or more than 8 MB for 60 seconds). Messages queued for that client are lost. See [Client handling](clients.md#output-buffer-limits)."

### F5 [minor] MISSING: Shard subscribers are unsubscribed when a slot moves (`topics/pubsub.md:178`)
**Doc says:**
> A shard message must be sent to a node that owns the slot the shard channel is hashed to.

**Problem:** The page does not say what happens to shard subscribers on resharding or failover. The server force-unsubscribes them, and clients must resubscribe on the new owner or they silently stop receiving messages.
**Evidence:** valkey@9.1.2:src/pubsub.c:369-377 `pubsubShardUnsubscribeAllChannelsInSlot`: "For each client subscribed to the channel, unsubscribe it." (sends an `sunsubscribe` push).
**Fix:** Add: "When a slot is migrated or the node loses ownership of it, Valkey sends an `sunsubscribe` message for each affected shard channel. Clients should resubscribe using the node that now owns the slot."

### F6 [nit] OUTDATED: Redis version used for sharded Pub/Sub (`topics/pubsub.md:177`)
**Doc says:**
> From Redis OSS 7.0, sharded Pub/Sub is introduced

**Problem:** Every Valkey release has it. Redis-era version note on a Valkey page.
**Fix:** "Sharded Pub/Sub assigns shard channels to slots using the same algorithm used for keys."

### F7 [nit] EXAMPLE: UNSUBSCRIBE reply order differs from 9.1.2 (`topics/pubsub.md:113`)
**Doc says:** reply lists `second :1` then `first :0`.
**Problem:** 9.1.2 replies `first :1` then `second :0` for the page's exact sequence. Order follows internal hash table iteration and is not guaranteed.
**Evidence:**
```
*3\r\n$11\r\nunsubscribe\r\n$5\r\nfirst\r\n:1\r\n*3\r\n$11\r\nunsubscribe\r\n$6\r\nsecond\r\n:0\r\n
```
**Fix:** Swap to match 9.1.2 output, and add: "The order in which channels are unsubscribed is not specified."

### F8 [nit] STYLE: Typos (`topics/pubsub.md:10`)
**Doc says:**
> If there are no subscribes to a channel ... persisted messages queues

line 42: "Valkey' Pub/Sub"
**Fix:** "subscribers", "persisted message queues", "Valkey Pub/Sub".

### F9 [nit] EXAMPLE: Wrong or missing fence language tags (`topics/pubsub.md:15`)
**Doc says:** line 15 ` ```bash ` around `SUBSCRIBE channel11 ch:00`; lines 77, 95, 101, 113, 136, 143, 162 have no tag.
**Problem:** These are Valkey commands and RESP wire output, not bash. Line 96 also uses a `> ` prompt that no other block uses.
**Fix:** Use ` ```text ` for the RESP blocks (77, 101, 113) and drop `bash` / tag command-only blocks consistently; remove the `> ` prefix on line 96.
